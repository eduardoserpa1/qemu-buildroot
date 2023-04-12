# Trabalho 1
José Eduardo Rodrigues Serpa (20200311)
Nicolle Canceri Lumertz (20103640)

Após concluirmos todas as etapas para a configuração do buildroot, devemos primeiramente, para iniciar o servidor http, estabelecer a conexão entre o guest e o host, para isso, devemos modificar um script 'S41network-config' encontrado na diretório custom_scripts.

Devemos alterar o campo <IP_DO_HOST> para o ip da maquina na qual está hospedando o buildroot. Para visualizar o ip da maquina host (como também na máquina guest), devemos executar o comando ifconfig.

![Screenshot from 2023-04-12 18-04-47](https://user-images.githubusercontent.com/47951275/231584814-efaa7650-b296-4e76-8802-8fb0162c32fc.png)

A linha 'inet', informa o ip que deve ser substituido pelo label <IP_DO_HOST> apresentado no trecho de código abaixo.

 ```
 #!/bin/sh
#
# Configuring host communication.
#

case "$1" in
  start)
	printf "Configuring host communication."
	
	/sbin/ifconfig eth0 192.168.1.10 up
	/sbin/route add -host <IP_DO_HOST> dev eth0
	/sbin/route add default gw <IP_DO_HOST>
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

Para que as modificações sejam aplicadas, será necessário recompilar a máquina guest. Para isso, execute o comando 'make'.

Após a execução do comando make, iremos executar o sistema para testar caso a conexão foi estabelecida com sucesso, para isso, execute a emulação da máquina guest com o seguinte comando

```
sudo qemu-system-i386 --device e1000,netdev=eth0,mac=aa:bb:cc:dd:ee:ff \
	--netdev tap,id=eth0,script=custom-scripts/qemu-ifup \
	--kernel output/images/bzImage \
	--hda output/images/rootfs.ext2 \
	--nographic \
	--append "console=ttyS0 root=/dev/sda" 
```

Dentro da máquina guest, teste a conexão com o host utilizando o comando 'ping', com alvo no ip da máquina host.

Caso o ping seja efetuado com sucesso, podemos iniciar a configuração do ambiente pyhton que irá iniciar o servidor http.

Antes de executarmos qualquer código python, será necessário executar o comando 'make menuconfig' para entrar nas configurações do buildroot e preparar o interpretador python.

ALERTA: Antes de fechar o meenu de configurações, não esqueça de salvar as alterações.

```
-- Target Packeges
    -- Interpreter languages and scripting
      [*] python3 
```

Em seguida, devemos também configurar o WCHAR, utilizando a seguinte configuração. Novamente iremos executar 'make menuconfig'.

```
 -- Toolchain
    C library (uClibc-ng) -->
    .
    .
    .
    [*] Enable WCHAR support
```

Após concluirmos as etapas anteriores, devemos novamente executar o comando 'make' para que as alterações sejam aplicadas com sucesso.

Agora que já preparamos todo ambiente para que possamos montar nosso servidor, iremos desenvolver e executar o código abaixo, qual é responsável por hospedar nosso servidor http, como também montar uma página HTML com informações básicas da máquina guest. 

```python

import os
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

Para editar códigos na máquina guest, devemos utilizar o editor de texto 'vi', com o comando 'vi <NOME_DO_ARQUIVO>.py'.
Em seguida, após salvarmos o código, iremos executa-lo utilizando o comando 'python <NOME_DO_ARQUIVO>.py'.

Contudo, deveremos obter um resultado semelhante ao print abaixo.

![Screenshot from 2023-04-12 18-24-49](https://user-images.githubusercontent.com/47951275/231588586-cb83a489-1910-4644-be0e-40556554c53a.png)

Com isso, temos certeza de que o servidor foi inciado com êxito, e poderá ser acessado via navegador na máquina host.

Para acessar a página HTML, devemos informar o IP da máquina guest na aba de pesquisa do navegador junto com a porta utilizada pelo servidor http do buildroot, como demonstra a imagem abaixo.

![Screenshot from 2023-04-12 17-47-15](https://user-images.githubusercontent.com/47951275/231582900-f4a72458-5f0a-4434-9fb1-2f0000815181.png)

Ao fim deste tutorial, deve ser possível visualizar a página HTML hospedada pelo buildroot através do servidor HTTP que criamos com script python, como também, deve ser possível visualizar diversas informações importantes do sistema como a lista de processos ativos na máquina guest.

