sudo mkdir -p /etc/systemd/system/docker.service.d

sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://127.0.0.1:10808"
Environment="HTTPS_PROXY=http://127.0.0.1:10808"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"

sudo systemctl daemon-reload
sudo systemctl restart docker
