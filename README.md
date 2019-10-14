dumps.rdb导出方式参考redis-cli文件

SQLite 分析内存快照
tools:
	pip install rdbtools python-lzf
	ucloud: http://redis-import-tool.ufile.ucloud.com.cn/uredis-redis-port

1、./uredis-redis-port dump -f     10.10.165.166:6379 -o save.rdb
2、rdb -c memory /redisfile/dump.rdb > test.csv
3、	sqlite3 rdb
	create table memory(database int,type varchar(128),key varchar(128),size_in_bytes int,encoding varchar(128),num_elements int,len_largest_element varchar(128), ex varchar(128));
	.mode csv memory  # sqlite mode
	.import test.csv memory  # 导入csv 到 sqlite
	select * from memory limit 10;
	delete from memory where ex !="";  # 删除有过期时间的key
	select * from memory order by size_in_bytes desc limit 10;  # 按内存使用大小
4、内存碎片率 = used_memory_rss / used_memory
	在 1 ～ 1.5间算正常，在大量删除key，redis 4.0 可以 memory purge 清理内存碎片，或者 config set activedefrag yes 自动清理碎片








2. 将redis的dump.rdb文件下载到本地, 用 rdbtools 工具生产内存报告

rdb -c memory /redisfile/dump.rdb > test.csv

sort -k4nr -t , test.csv > sort.txt

# 查看前1000个排序最高的数据
awk -F ',' '{print substr($3, 0,18)}' sort.txt | head -1000 | sort -k1 | uniq

# 查看sort.txt的结果，一般能得出类似'my_rank_top'开头的集合占用最高，排在了前面。若要查看类似'my_rank_top'开头的key总共占用了多少内存，可以用命令：
cat sort.txt | grep 'my_rank_top' | awk -F ',' '{sum += $4};END {print sum}'

# 模糊匹配key来删除
redis-cli -h 127.0.0.1 -p 6379  keys 'my_ranking_list*' | xargs redis-cli -h 127.0.0.1 -p 6379 del





3、scan

import logging
import sys
import redis
import datetime

BigNum = 10000
string_keys = []
hash_keys = []
list_keys = []
set_keys = []
zset_keys = []


def scan_hash(source):
    for key in hash_keys:
        key = key.decode()
        hlen = source.hlen(key)
        if hlen > BigNum:
            logging.error("Hash key:%s, size:%d" % (key, hlen))


def scan_set(source):
    for key in set_keys:
        key = key.decode()
        slen = source.scard(key)
        if slen > BigNum:
            logging.error("Set key:%s, size:%d" % (key, slen))


def scan_zset(source):
    for key in zset_keys:
        key = key.decode()
        zlen = source.zcard(key)
        if zlen > BigNum:
            logging.error("ZSet key:%s, size:%d" % (key, zlen))


def scan_list(source):
    for key in list_keys:
        key = key.decode()
        llen = source.llen(key)
        if llen > BigNum:
            logging.error("Hash key:%s, size:%d" % (key, llen))


def read_type_keys(source, ScanIndex):
    ScanIndex, keys = source.execute_command(
        'scan', ScanIndex, "count", 200000)
    keys_count = len(keys)
    ScanIndex = ScanIndex.decode()
    # print("scanIndex: %s  keyCount: %s" % (ScanIndex, keys_count))
    pipe = source.pipeline(transaction=False)
    index = 0
    pipe_size = 5000
    while index < keys_count:
        old_index = index
        num = 0
        while (index < keys_count) and (num < pipe_size):
            pipe.type(keys[index])
            index += 1
            num += 1
        results = pipe.execute()
        for tp in results:
            tp = tp.decode()
            if tp == "string":
                string_keys.append(keys[old_index])
            elif tp == "list":
                list_keys.append(keys[old_index])
            elif tp == "hash":
                hash_keys.append(keys[old_index])
            elif tp == "set":
                set_keys.append(keys[old_index])
            elif tp == "zset":
                zset_keys.append(keys[old_index])
            elif tp == "none":
                pass
            else:
                logging.error("no key: %s   type: %s" % (keys[old_index], tp))
            old_index += 1
    return ScanIndex


if __name__ == '__main__':
    argc = len(sys.argv)
    if argc < 3:
        logging.error("usage: %s sourceIP sourcePort [password]" % (sys.argv[0]))
        exit(1)
    SrcIP = sys.argv[1]
    SrcPort = int(sys.argv[2])
    Password = ""
    if argc == 4:
        Password = sys.argv[3]

    start = datetime.datetime.now()
    source = redis.Redis(host=SrcIP, port=SrcPort, password=Password)

    index = 0
    times = 1
    strNum = 0
    setNum = 0
    zsetNum = 0
    listNum = 0
    hashNum = 0
    logging.error("begin scan key")
    while index != '0' or times == 1:
        first = False
        logging.error("Times: %s" % times)
        times += 1
        index = read_type_keys(source, index)
        strNum += len(string_keys)
        setNum += len(set_keys)
        zsetNum += len(zset_keys)
        listNum += len(list_keys)
        hashNum += len(hash_keys)

        scan_hash(source)
        scan_list(source)
        scan_set(source)
        scan_zset(source)
        string_keys = []
        hash_keys = []
        list_keys = []
        set_keys = []
        zset_keys = []

    logging.error("String Key Count is: %s " % strNum)
    logging.error("Set Key Count is :   %s " % setNum)
    logging.error("ZSet Key Count is:   %s " % zsetNum)
    logging.error("List Key Count is:   %s " % listNum)
    logging.error("Hash Key Count is:   %s " % hashNum)

    stop = datetime.datetime.now()
    diff = stop - start
    logging.error("Finish, token time: %s" % str(diff))
