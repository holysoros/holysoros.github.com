Linux 开发时，通常会有如下几种需求：


- 查看某个进程打开着哪些 file/socket 等;

    `lsof -p <pid> -P`

- 查看某个 file/socket 等被哪个进程打开着;

    `lsof -p <file>`

- 查看当前系统中所有被打开着的 file/socket ;


    -i
    [46][protocol][@hostname|hostaddr][:service|port]

    lsof -i4udp:4000
