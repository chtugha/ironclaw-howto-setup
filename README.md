# ironclaw-howto-setup
This is a guide how to setup ironclaw on a minimal debian system and to connect it to a local ollama instance. 


If not already running: This is how you set up Ollama on a debian console:


curl -fsSL https://ollama.com/install.sh | sh


ollama pull qwen3:14b

ollama run qwen3:14b


Your service file (usually at: /etc/systemd/system/ollama.service) should look like this if you want the model to be loaded on reboot:

<pre>[Unit]
#Description=Ollama Service
#After=network-online.target

[Service]
#ExecStart=/usr/local/bin/ollama serve
#User=ollama
#Group=ollama
#Restart=always
#RestartSec=3
#Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_KEEP_ALIVE=-1"
Environment="OLLAMA_DEBUG=0"
Environment="OLLAMA_CONTEXT_LENGTH=8192"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q4_0"
Environment="OLLAMA_NUM_THREAD=4"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_NO_CLOUD=1"
Environment="OLLAMA_NUM_PARALLEL=1"
ExecStartPost=/bin/sh -c 'until curl -s http://localhost:11434 > /dev/null; do sleep 1; done'
ExecStartPost=/usr/bin/curl -s http://localhost:11434/api/generate -d '{"model":"voytas26/openclaw-qwen3vl-8b-opt:latest","prompt":"hi","stream":false}'

[Install]
#WantedBy=default.target</pre>



If you are running Ollama and ironclaw on the same machine you don't need the nginx step. If you don't run Ollama on the same Machine ironclaw will refuse to connect to Ollama without TLS-Encryption.
Because we don't want to fiddle with Certificates and such for now we will route the connection trough nginx and trick ironclaw into accepting an insecure connection to Ollama like that. Don't do that at home kids!


Installing nginx: DO NOT FORGET TO EXCHANGE X.X.X.X FOR YOUR DESIRED IP


add this:  
nano /etc/nginx/sites-available/ollama
<pre>
server {
    listen 11434;
    server_name ollama.local;

    location / {
        proxy_pass http://X.X.X.X:11434;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Connection "";
        proxy_buffering off;

        # LLM generation can take minutes
        proxy_read_timeout 600s;
    }
}
    
</pre>

ln -s /etc/nginx/sites-available/ollama /etc/nginx/sites-enabled/
    
nginx -t
    
systemctl daemon-reload
    
systemctl restart nginx



Installing Rust:

apt-get install curl build-essential gcc make rustc -y


Installing Postgres:

apt-get install postgresql postgresql-contrib -y
    
apt-get install postgresql-server-dev-17 postgresql-17-pgvector -y (the number might have changed as you read this - 17 is from 03/2026)

nano /etc/postgresql/17/main/postgresql.conf

<pre>
uncomment #listen_addresses = 'localhost'
</pre>

systemctl daemon-reload

systemctl restart postgres.service

systemctl restart postgres@17-main.service    (coud vary...find out with tab)


    
Installing ironclaw:

curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/nearai/ironclaw/releases/latest/download/ironclaw-installer.sh | sh

source $HOME/.cargo/env



Creating the Database:

sudo -i -u postgres psql -c "CREATE ROLE captainawesome LOGIN SUPERUSER;"     (You can of course exchange captainawesome for whatever you want)

sudo -i -u postgres createdb ironclaw
    
sudo -i -u postgres psql -d ironclaw -c "CREATE EXTENSION IF NOT EXISTS vector;"
    
sudo -i -u postgres psql -c "ALTER USER captainawesome WITH PASSWORD '1337';"

    
Configure ironclaw with the onboard wizard:

Execute this command:
    
ironclaw onboard

<pre>Make these selections:
1 for PostgreSQL
Database URL: postgres://captainawesome:1337@localhost:5432/ironclaw?sslmode=disable
Run database migrations? Y
3 Skip
4 Ollama
Ollama base URL: http://X.X.X.X:11434
Select a model: choose "qwen3:14b"
Enable semantic search? Y
Configure a tunnel? N
Which channels do you want to enable? Check HTTP webhook. Check others as you wish.
Which tools do you want to install? Make sure Gmail is checked. Check others as you wish.
Enable a sandbox? N
Enable heartbeat? N</pre>
    

Now Configure the Gateway and the Webhooks:

cd /

find / -type f -name ".env" 2>/dev/null

nano /root/.ironclaw/.env    (or wherever you find the .env file)

<pre>
GATEWAY_HOST=X.X.X.X (set 0.0.0.0 for any interface, or set a distinct interface IP like 192.168.1.11)
GATEWAY_PORT=3000    
HTTP_WEBHOOK_SECRET="12345"
EMBEDDING_ENABLED=true
EMBEDDING_DIMENSION=384
</pre>
    
And run ironclaw:

apt-get install screen

screen (hit enter on the blabla screen, it says that you just started a virtual terminal)

ironclaw run

connect to the web-ui with your browser by copying the link on the screen (Exchange 0.0.0.0 for the actual IP)

connect to the web-ui with the link on your screen

Press CTRL+A followed by CTRL+D     (You can reaatach to the virtual terminal by typing: screen -r)
