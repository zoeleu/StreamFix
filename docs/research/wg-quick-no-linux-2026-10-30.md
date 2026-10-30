# O contrato do `wg-quick` no Linux

Medição de 30/10/2026, para habilitar o Linux no instalador de shell. Máquina: Arch Linux, kernel
7.2 (zen), `wireguard-tools v1.0.20260223` e `openresolv` pelo `pacman`. Feita com o
`controle-wg.ts` de produção, montado com os caminhos de Linux de `native.ts`, contra um perfil
descartável (`sfx-medicao`, `AllowedIPs = 192.0.2.0/24`, endpoint TEST-NET-1, sem `DNS`).

**O que esta medição não alcança:** Discord, Go Live, se o gate aceita a mídia, latência real, e
outras distribuições (Debian/Ubuntu/Fedora/openSUSE estão no instalador por nome de pacote, não
por medição).

## 1. O `/etc/wireguard` é só-root

```
drwx------ 1 root root 0 Feb 24  2026 /etc/wireguard
ls: cannot open directory '/etc/wireguard': Permission denied
```

No macOS ele é `drwxr-xr-x`, e o `existe()` listava sem privilégio. Aqui isso responderia
`desconhecido` para sempre, e o plugin ficaria no completo a sessão inteira.

**Consequência.** `lsComSudo` em `controle-wg.ts` e `/usr/bin/ls -1 /etc/wireguard` exato na
regra de sudoers. Medido: `existe("sfx-medicao") = true`, `existe("nao-existe") = false`.

## 2. O resto do contrato é o mesmo do Darwin

- `wg show interfaces` devolve o nome do perfil (`sfx-medicao\n`) — no Linux a interface leva o
  nome do perfil, então a identificação por destinos continua funcionando, só que sem precisar.
- `wg show <iface> allowed-ips` e `latest-handshakes`: duas colunas, `chave<TAB>valor`, igual ao
  achado 14 do Darwin.
- `wg-quick up` devolve 0 com o peer morto (achado 4). O `subir` do controle recusou com
  "não fechou handshake em 2000 ms" e derrubou a interface; `estado` voltou a `fora`.
- `down` de quem já está fora: `` `sfx-medicao' is not a WireGuard interface ``, como no Darwin.
- O limite de 15 caracteres é a mesma linha do mesmo script.

## 3. O `DNS =` do perfil exige `resolvconf`

O `wg-quick` do Linux aplica `DNS =` chamando `resolvconf`, e nenhuma distribuição grande o traz
por padrão: sem ele o `up` falha inteiro. O Arch lista o `openresolv` como dependência opcional
do `wireguard-tools` ("for DNS functionality"). O instalador instala o `openresolv`, ou o shim do
`systemd-resolved` quando ele está no ar.

**Em aberto:** quem gerencia o `/etc/resolv.conf` por fora (o Tailscale nesta máquina) pode
sobrescrever o DNS do túnel. O instalador avisa; não conserta.

## 4. A regra de sudoers passa no `visudo`

```
zoe ALL=(root) NOPASSWD: /usr/bin/bash /usr/bin/wg-quick up streamfix, /usr/bin/bash /usr/bin/wg-quick down streamfix, /usr/bin/bash /usr/bin/wg-quick up streamfix-ctl, /usr/bin/bash /usr/bin/wg-quick down streamfix-ctl, /usr/bin/wg show *, /usr/bin/ls -1 /etc/wireguard
```

## O que continua em aberto

- Uma transmissão aceita de ponta a ponta, com uma saída de verdade.
- O IPv6: os perfis só carregam IPv4, e numa rede com IPv6 o Discord pode abrir o gateway por
  fora do túnel. Vale nos três sistemas, e nunca foi medido.
- O Discord de flatpak por `flatpak-spawn --host`.
- Corte por aplicativo no Linux (cgroup + `fwmark`), que o macOS não tem como fazer e o Linux tem.
