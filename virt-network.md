# Configuração de Rede Bridge no KVM/Libvirt (Ubuntu)

## Problema

A VM no KVM/Libvirt:
- iniciava normalmente
- recebia IP da rede NAT (`192.168.122.x`)
- mas não tinha acesso à internet

Mesmo após:
- reinstalar `libvirt`
- configurar `firewalld`
- habilitar `masquerade`
- ajustar `FORWARD`
- reiniciar `libvirtd`

O ambiente continha:
- Docker
- Netbird/Wireguard
- VPN (`tun0`)
- firewalld
- NetworkManager

Isso causava conflito no NAT/forwarding do `libvirt`.

---

# Solução

Foi utilizada uma bridge real (`br0`) conectada diretamente à interface física `enp5s0`.

Assim:
- a VM passou a receber IP diretamente do roteador
- eliminando problemas de NAT do libvirt

---

# Passo a Passo

## 1. Criar a bridge

```bash
nmcli connection add type bridge ifname br0 con-name br0
```

---

## 2. Adicionar interface física à bridge

```bash
nmcli connection add type bridge-slave ifname enp5s0 master br0
```

---

## 3. Configurar DHCP na bridge

```bash
nmcli connection modify br0 ipv4.method auto
nmcli connection modify br0 ipv6.method auto
```

---

## 4. Subir a bridge

```bash
sudo nmcli connection up br0
```

---

## 5. Associar interface física manualmente (caso necessário)

Se a bridge permanecer em estado:

```text
NO-CARRIER
```

executar:

```bash
sudo ip link set enp5s0 master br0
sudo ip link set br0 up
```

---

## 6. Verificar bridge

```bash
ip a
```

Resultado esperado:

```text
br0: <BROADCAST,MULTICAST,UP,LOWER_UP>
inet 192.168.x.x
```

---

# Configuração da VM no Virt-Manager

## Alterar placa de rede

Na VM:

```text
Hardware → Placa de Rede
```

Configurar:

### Fonte da rede

```text
Bridge device
```

### Nome do dispositivo

```text
br0
```

### Modelo da placa

```text
e1000e
```

---

# Resultado

A VM passou a:
- obter IP diretamente do roteador
- acessar internet normalmente
- funcionar como dispositivo real da rede local

Exemplo:

```text
192.168.10.x
```

---

# Verificações úteis

## Ver conexões do NetworkManager

```bash
nmcli connection show
```

## Ver interfaces

```bash
ip a
```

## Ver bridge

```bash
bridge link
```

---

# Observações

Ambientes contendo:
- Docker
- Wireguard
- Netbird
- VPN
- firewalld
- múltiplas bridges

podem causar problemas no NAT padrão do `libvirt`.
