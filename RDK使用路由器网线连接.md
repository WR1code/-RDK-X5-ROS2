sudo dhclient -v -r eth0   # 释放旧的
sudo dhclient -v eth0      # 重新获取
ifconfig重新查询IP，一般eth0为有线名称，wlan为无线
