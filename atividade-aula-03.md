
Nome: João Daniel Pereira da Silva
Matricula: 2024338916
Turma: 913a
Data: 26/10/2026


---

## Passo 1: Mapeamento de Interfaces, IP, Máscara, MAC e Gateway

A partir da execução dos comandos `ip addr show`, `ip route` e `netplan status` na máquina virtual, identificamos os seguintes parâmetros de rede:

| Parâmetro | Valor Objeto | Comando de Origem |
| :--- | :--- | :--- |
| **Nome da Interface** | `enp0s3` | `ip addr show` / `netplan status` |
| **Endereço IP e Prefixo** | `10.0.2.15/24` | `ip addr show` (`inet 10.0.2.15/24`) |
| **Máscara de Rede** | `255.255.255.0` (corresponde ao `/24`) | Calculado a partir da notação CIDR |
| **Endereço Físico (MAC)** | `08:00:27:d3:24:65` | `ip addr show` (`link/ether`) |
| **Gateway Padrão** | `10.0.2.2` | `ip route` (`default via 10.0.2.2`) |
| **Servidor DNS** | `10.0.2.3` | `netplan status` |

---

## Passo 2: Testes de Conectividade em Camadas

Executamos os testes de conectividade de forma progressiva (do ponto mais próximo ao mais distante):

1. **Teste de Loopback (Pilha TCP/IP Local):**
   ```bash
   ping -c 4 127.0.0.1
   ```
   *Resultado:* `4 packets transmitted, 4 received, 0% packet loss` (Sucesso).

2. **Teste da Própria Interface (IP da VM):**
   ```bash
   ping -c 4 10.0.2.15
   ```
   *Resultado:* `4 packets transmitted, 4 received, 0% packet loss` (Sucesso).

3. **Teste do Gateway Padrão (Roteador Virtual):**
   ```bash
   ping -c 4 10.0.2.2
   ```
   *Resultado:* `4 packets transmitted, 4 received, 0% packet loss` (Sucesso).

4. **Teste de Conectividade Externa via IP (Internet):**
   ```bash
   ping -c 4 8.8.8.8
   ```
   *Resultado:* `4 packets transmitted, 4 received, 0% packet loss, time 3024ms` (Sucesso).

5. **Teste de Resolução de Nomes (DNS):**
   ```bash
   ping -c 4 google.com
   ```
   *Resultado:* Nome resolvido para o IP `172.217.30.14`, `0% packet loss` (Sucesso).

---

## Passo 3: Diagnóstico da Camada de Falha

Analisando a sequência de testes em camadas:
* Todas as 5 etapas retornaram **0% de perda de pacotes** (`0% packet loss`).
* **Conclusão:** Não há falhas na saída de dados da Máquina Virtual. As camadas de Aplicação, Transporte, Rede, Enlace e Física estão totalmente operacionais para tráfego saintes (*outbound*).

---

## Passo 4: Teste de Acessibilidade Host -> VM (Modo NAT)

Para testar o acesso do computador físico (Host) para a Máquina Virtual em modo NAT, executamos no terminal da máquina hospedeira:

```bash
ping 10.0.2.15
```

* **Resultado Observado:** O ping falha (Timeout / Request Timed Out / Destination Host Unreachable).
* **Evidência:** O Host não consegue enviar pacotes diretamente para o IP `10.0.2.15`.

---

## Passo 5: Explicação Teórica do Funcionamento do NAT

O modo **NAT (Network Address Translation)** no VirtualBox cria uma rede privada virtualizada isolada para a VM:

1. **Tráfego VM $\rightarrow$ Internet:** A VM consegue navegar pois o hipervisor traduz o IP interno (`10.0.2.15`) para o IP da interface física do Host antes de enviar o pacote para a rede externa.
2. **Tráfego Host $\rightarrow$ VM:** O IP `10.0.2.15` faz parte de uma sub-rede virtual interna gerenciada apenas pelo VirtualBox. A máquina física (Host) não possui rota para essa rede nem reconhece o IP `10.0.2.15` como um destino válido no seu barramento de rede local, tornando a VM invisível para conexões de fora para dentro (sem redirecionamento de portas).

---

## Passo 6: Alteração para Modo Bridge (Ponte)

Ao alterar a interface de rede da VM nas configurações do VirtualBox de **NAT** para **Placa em Modo Bridge (Bridge)**:

1. A VM passa a atuar como um dispositivo físico independente conectado diretamente à placa de rede do hospedeiro.
2. Execução do comando `ip a` na VM após a alteração:
   * **Novo IP recebido via DHCP local:** `[Preencher com o novo IP gerado ex: 192.168.1.X]`
3. Execução do teste de ping do Host para o novo IP da VM:
   ```bash
   ping [Novo-IP-em-Bridge]
   ```
   * **Resultado:** Resposta com sucesso (`0% packet loss`), demonstrando visibilidade total na rede local.

---

## Passo 7: Comparativo Evidenciado: NAT vs. Bridge

| Critério | Modo NAT | Modo Bridge (Ponte) |
| :--- | :--- | :--- |
| **Faixa de IP da VM** | Faixa Privada Virtual (`10.0.2.15/24`) | Faixa da Rede Física Local (ex: `192.168.X.X`) |
| **Acesso da VM para a Internet** | Sim (via NAT do Hospedeiro) | Sim (direto pelo Roteador Físico) |
| **Acesso do Host para a VM** | **Não** (requer redirecionamento de portas) | **Sim** (comunicação direta via IP) |
| **Outros computadores da rede veem a VM?** | Não | Sim |