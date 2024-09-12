一、写一个定时执行的Bash脚本，**每月的一号凌晨1点** 对 MongoDB 中 test.user_logs 表进行备份、清理
  - 首先备份上个月的数据，备份完成后打包成.gz文件
  - 备份文件通过sfpt传输到 **Backup [bak@bak.ipo.com]** 服务器上，账户已经配置在~/.ssh/config;
  - 备份完成后，再对备份过的数据进行清理: **create_on [2024-01-01 03:33:11]** ;
  - 如果脚本执行失败或者异常，则调用 [https://monitor.ipo.com/webhook/mongodb ];
  - 这个表每日数据量大约在 **200w** 条, 单条数据未压缩的存储大小约 **200B**;


Q:
- 上个月的数据以什么字段判断日期, create_on?
- fix typo: sfpt -> sftp
- 调用 webhook 使用哪种 http request methods, GET or POST，需要传递什么参数吗
- 单日数据量 200w * 200B = 400 MB/day, 月数据量 400MB * 30 = 12 GB/month


Solution:

`crontab` 设置定时任务执行备份脚本

```
# 每月一号凌晨1点执行 mongodb 备份脚本
0 1 1 * * /usr/local/bin/mongodb_backup.sh
```

mongodb_backup.sh

```bash
#!/usr/bin/env bash

DB_NAME="test"
COLLECTION="user_logs"
BACKUP_DIR="/opt/mongodb/backup" # 备份的本地路径
REMOTE_BACKUP_DIR="/opt/mongodb/backup" # 备份的远程服务器的路径
BACKUP_DATE=$(date -d 'last month' +%Y-%m) # 备份上个月的数据
BACKUP_FILE="$BACKUP_DIR/user_logs_$BACKUP_DATE.json.gz"
SFTP_SERVER="bak@bak.ipo.com"
WEBHOOK_URL="https://monitor.ipo.com/webhook/mongodb"
LOG_FILE="/var/log/mongodb_backup.log"

# 导出 MongoDB 上个月数据
backup_data() {
    mongodump --db $DB_NAME --collection $COLLECTION \
        --query \
        "{\"create_on\": 
            {
                \"\$gte\": {\"\$date\": \"$BACKUP_DATE-01T00:00:00Z\"},
                \"\$lt\": {\"\$date\": \"$(date +%Y-%m)-01T00:00:00Z\"}
            }
        }" \
        --gzip --out $BACKUP_FILE
    
    if [ $? -ne 0 ]; then
        echo "Backup data failed" >> $LOG_FILE
        curl -X GET $WEBHOOK_URL
        exit 1
    else
        echo "Backup success: $BACKUP_FILE" >> $LOG_FILE
    fi
}

# 将备份数据传输到远程服务器
transfer_backup() {
    sftp $SFTP_SERVER <<EOF
    put $BACKUP_FILE $REMOTE_BACKUP_DIR/
EOF

    if [ $? -ne 0 ]; then
        echo "Transfer backup failed" >> $LOG_FILE
        curl -X GET $WEBHOOK_URL
        exit 1
    else
        echo "Transfer backup success: $BACKUP_FILE" >> $LOG_FILE
    fi
}

# 清理备份过的数据
clean_data() {
    mongo $DB_NAME --eval \
        "db.$COLLECTION.deleteMany(
            {
                create_on: {
                    \"\$gte\": new Date(\"$BACKUP_DATE-01T00:00:00Z\"), 
                    \"\$lt\": new Date(\"$(date +%Y-%m)-01T00:00:00Z\")
                }
            }
        )"

    if [ $? -ne 0 ]; then
        echo "Clean data failed" >> $LOG_FILE
        curl -X GET $WEBHOOK_URL
        exit 1
    else
        echo "Clean data success" >> $LOG_FILE
    fi
}

main() {
    echo "Start backup: $(date)" >> $LOG_FILE
    backup_data
    transfer_backup
    clean_data
    echo "Backup and Clean up finished: $(date)" >> $LOG_FILE
}

main

```