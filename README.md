# Trabalho 1
José Eduardo Rodrigues Serpa (20200311)
Nicolle Canceri Lumertz (20103640)



```python

iimport os
import sys
import time
import socket
from http.server import BaseHTTPRequestHandler,HTTPServer

HOST_NAME = '192.168.1.10' # !!!REMEMBER TO CHANGE THIS!!!
PORT_NUMBER = 8000

def get_time():
    return os.popen("date").read()

def get_uptime():
    with open('/proc/uptime', 'r') as f:
        uptime_seconds = float(f.readline().split()[0])
        return uptime_seconds

def get_cpu_info():
    with open('/proc/cpuinfo', 'r') as f:
        cpuinfo = f.readlines()

    model_name = [line.split(':')[-1].strip() for line in cpuinfo if 'model name' in line][0]
    mhz = [line.split(':')[-1].strip() for line in cpuinfo if 'cpu MHz' in line][0]
    return model_name, mhz

def get_cpu_percent():
    with open('/proc/stat', 'r') as f:
        fields = [float(field) for field in f.readline().strip().split()[1:]]
        idle_time = fields[3]
        total_time = sum(fields)
        return (1.0 - idle_time / total_time) * 100.0

def get_mem_info():
    with open('/proc/meminfo', 'r') as f:
        meminfo = f.readlines()

    total_memory = int([line.split(':')[-1].strip() for line in meminfo if 'MemTotal' in line][0].split()[0]) // 1024

    free = int([line.split(':')[-1].strip() for line in meminfo if 'MemFree' in line][0].split()[0])

    buffers = int([line.split(':')[-1].strip() for line in meminfo if 'Buffers' in line][0].split()[0])

    cached = int([line.split(':')[-1].strip() for line in meminfo if 'Cached' in line][0].split()[0])

    used_memory =   ((total_memory - free - buffers - cached) // 1024) * -1

    return total_memory, used_memory

def get_processes():
    processes = []
    for pid in os.listdir('/proc'):
        try:
            pid = int(pid)
            with open(os.path.join('/proc', str(pid), 'stat'), 'rb') as f:
                cmdline = f.read().decode().split(" ")
                if cmdline:
                    processes.append({'pid': cmdline[0], 'name': cmdline[1].strip("()")})
        except (ValueError, FileNotFoundError):
            pass
    return processes

def get_system_info():
    hostname = socket.gethostname()
    os_name = os.uname().sysname
    os_version = os.uname().release

    return hostname, os_name, os_version

def format_time(seconds):
    m, s = divmod(seconds, 60)
    h, m = divmod(m, 60)
    return f"{h:.2f}:{m:.2f}:{s:.2f} ."

def format_processes(processes):
    return '\n'.join([f"{p['pid']}: {p['name']}" for p in processes])

def generate_html():
    uptime = get_uptime()
    cpu_model, cpu_mhz = get_cpu_info()
    cpu_percent = get_cpu_percent()
    total_memory, used_memory = get_mem_info()
    processes = get_processes()
    hostname, os_name, os_version = get_system_info()

    html = f"""
    <html>
        <head>
            <title>Informações do Sistema</title>
            <meta charset="UTF-8">
        </head>
        <body>
            <h1>Informações do Sistema</h1>
            <h3>José Eduardo Rodrigues Serpa e Nicolle Canceri Lumertz</h3>
            <ul>
                <li>Data e hora do sistema: {get_time()}</li>
                <li>Tempo de funcionamento sem reinicialização do sistema:{format_time(uptime)}</li>
                <li>Modelo do processador: {cpu_model}</li>
                <li>Velocidade do processador: {cpu_mhz} MHz</li>
                <li>Capacidade ocupada do processador: {cpu_percent:.2f}%</li>
                <li>Quantidade de memória RAM total: {total_memory} MB</li>
                <li>Quantidade de memória RAM usada: {used_memory} MB</li>
                <li>Versão do sistema: {os_name} {os_version}</li>
                <li>Lista de processos em execução: <pre>{format_processes(processes)}</pre></li>
            </ul>
        </body>
    </html>
    """
    return html





class MyHandler(BaseHTTPRequestHandler):
    def do_HEAD(s):
        s.send_response(200)
        s.send_header("Content-type", "text/html")
        s.end_headers()
    def do_GET(s):
        """Respond to a GET request."""
        s.send_response(200)
        s.send_header("Content-type", "text/html")
        s.end_headers()
        #s.wfile.write("<html><head><title>Title goes here.</title></head>".encode())
        #s.wfile.write("<body><p>This is a test.</p>".encode())
        # If someone went to "http://something.somewhere.net/foo/bar/",
        # then s.path equals "/foo/bar/".
        #s.wfile.write(f"<p>You accessed path: {s.path}</p>".encode())
        s.wfile.write(generate_html().encode())

if __name__ == '__main__':
    httpd = HTTPServer((HOST_NAME, PORT_NUMBER), MyHandler)
    print("Server Starts - %s:%s" % (HOST_NAME, PORT_NUMBER))
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        pass
    httpd.server_close()
    print("Server Stops - %s:%s" % (HOST_NAME, PORT_NUMBER))
```
![Screenshot from 2023-04-12 17-47-15](https://user-images.githubusercontent.com/47951275/231582900-f4a72458-5f0a-4434-9fb1-2f0000815181.png)
 file:///home/ze/Pictures/Screenshots/Screenshot%20from%202023-04-12%2017-47-39.png
 
 
 ```
 #!/bin/sh
#
# Configuring host communication.
#

case "$1" in
  start)
	printf "Configuring host communication."
	
	/sbin/ifconfig eth0 192.168.1.10 up
	/sbin/route add -host 10.115.240.143 dev eth0
	/sbin/route add default gw 10.115.240.143
	[ $? = 0 ] && echo "OK" || echo "FAIL"
	;;
  stop)
	printf "Shutdown host communication. "
	/sbin/route del default
	/sbin/ifdown -a
	[ $? = 0 ] && echo "OK" || echo "FAIL"
	;;
  restart|reload)
	"$0" stop
	"$0" start
	;;
  *)
	echo "Usage: $0 {start|stop|restart}"
	exit 1
esac

exit $?

```

