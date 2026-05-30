# router-internet:

## Salir a internet desde cualquier equipo de la red

    Instalar iptables-persistent

    iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE

    sudo netfilter-persistent save

# 8 - configurar todos los routers para que usen el DNS nuestro

    systemctl disable --now systemd-resolved       # si esta activo va a sobreescribir a /etc/resolv.conf

    nameserver 192.168.152.1
    search redes.cs.intra



# 10 - Verificar las reversas de todas las ips de la topologia

    for ip in \
    192.168.148.1 192.168.148.2 \
    192.168.150.1 \
    192.168.151.1 192.168.151.129 192.168.151.130 \
    192.168.151.193 \
    192.168.152.1 192.168.152.65 192.168.152.66 \
    192.168.152.69 192.168.152.70 \
    172.31.0.1 172.31.0.254
    do
    echo -n "$ip -> "
    dig @192.168.152.1 -x $ip +short
    done