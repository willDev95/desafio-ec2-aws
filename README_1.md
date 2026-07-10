# Gerenciamento de Instâncias EC2 na AWS

Repositório de anotações e insights construído durante o laboratório prático de **Gerenciamento de Instâncias EC2**, parte da trilha de estudos da [DIO (Digital Innovation One)](https://www.dio.me/).

O objetivo aqui não é só "seguir o passo a passo", mas documentar o raciocínio por trás de cada decisão ao provisionar, configurar e gerenciar uma instância EC2 real na AWS.

---

## 📋 Sumário

- [Sobre o desafio](#-sobre-o-desafio)
- [O que é o Amazon EC2](#-o-que-é-o-amazon-ec2)
- [Conceitos praticados](#-conceitos-praticados)
- [Passo a passo prático realizado](#-passo-a-passo-prático-realizado)
- [Comandos utilizados](#-comandos-utilizados)
- [Resultado do experimento: IP dinâmico](#-resultado-do-experimento-ip-dinâmico)
- [Desafios encontrados](#-desafios-encontrados)
- [Boas práticas aprendidas](#-boas-práticas-aprendidas)
- [Imagens](#-imagens)
- [Referências](#-referências)

---

## 🎯 Sobre o desafio

Este repositório é o entregável do desafio de projeto **"Gerenciando Instâncias EC2 da Amazon"**, cujo objetivo é consolidar, na prática, os conceitos fundamentais de computação em nuvem usando o serviço EC2 (Elastic Compute Cloud) da AWS.

Todo o processo abaixo foi executado de ponta a ponta em uma conta AWS real, dentro do Free Tier.

---

## ☁️ O que é o Amazon EC2

O **Amazon EC2** é o serviço de computação em nuvem da AWS que permite provisionar servidores virtuais (instâncias) sob demanda, com capacidade escalável, pagando apenas pelo que é utilizado (modelo *pay-as-you-go*).

Na prática, é um computador remoto que você "aluga" por tempo de uso — útil para hospedar aplicações, rodar processamento sob demanda, ambientes de teste, ou escalar infraestrutura conforme a demanda, sem precisar comprar e manter hardware físico.

---

## 🧠 Conceitos praticados

### 1. AMI (Amazon Machine Image)
Modelo/imagem usada para lançar a instância. Utilizei a **Amazon Linux 2023**, imagem oficial da AWS, já otimizada para rodar no EC2 e elegível para o nível gratuito.

### 2. Tipo de instância
Usei `t3.micro` — 2 vCPUs, 1 GiB de RAM, instância de uso geral com capacidade de *burst*, qualificada para o Free Tier.

### 3. Key Pair (par de chaves)
Criei um par de chaves RSA no formato `.pem`, baixado localmente. É essa chave privada que autentica o acesso via SSH — sem senha, apenas criptografia assimétrica. Reforço importante: **o arquivo `.pem` nunca deve subir para o repositório do GitHub**, pois equivale a uma senha de acesso.

### 4. Security Groups
Configurei o Security Group (`launch-wizard-1`) liberando a porta **22 (SSH)** apenas para o meu próprio IP (`Meu IP`), em vez de liberar para `0.0.0.0/0` (toda a internet). As portas HTTP (80) e HTTPS (443) ficaram fechadas, já que não havia necessidade de expor um serviço web.

### 5. VPC e Sub-rede
A instância foi lançada na VPC padrão da conta, com atribuição automática de IP público habilitada — o que permitiu o acesso externo via SSH.

### 6. Volumes EBS (Elastic Block Store)
Volume raiz de **8 GiB**, tipo `gp3` (SSD de propósito geral), anexado automaticamente como disco da instância.

### 7. Estados da instância
Testei na prática os estados **Running** (executando) e **Stopped** (interrompida), usando as ações "Parar instância" e "Iniciar instância" — sem nunca confundir com "Encerrar instância" (Terminate), que é destrutivo e irreversível.

### 8. IP público dinâmico vs. Elastic IP
Esse foi o insight mais valioso do laboratório (detalhado [abaixo](#-resultado-do-experimento-ip-dinâmico)): o IP público padrão do EC2 **muda** a cada ciclo de parar/iniciar. Para manter um IP fixo, seria necessário associar um **Elastic IP**.

### 9. Conexão via SSH
Conectei à instância via terminal (Git Bash no Windows), usando o cliente SSH nativo, sem depender do conector web da AWS (EC2 Instance Connect).

---

## 🛠️ Passo a passo prático realizado

1. **Login na conta AWS** e acesso ao console EC2.
2. **Launch Instance**:
   - Nome: `desafio-ec2-dio`
   - AMI: Amazon Linux 2023 (`ami-002192a70217ac181`)
   - Tipo: `t3.micro`
   - Par de chaves: criado novo, formato `.pem`
   - Rede: VPC padrão, IP público automático habilitado
   - Security Group: novo grupo (`launch-wizard-1`), SSH liberado apenas para "Meu IP"
   - Armazenamento: 8 GiB, `gp3`
3. **Instância lançada com sucesso** (ID: `i-0a38593df451c0de3`).
4. **Conexão via SSH** pelo terminal local, usando o IP público gerado.
5. **Validação do ambiente** dentro da instância (usuário, sistema operacional, disco).
6. **Teste de parar e iniciar a instância**, observando a mudança do IP público.

---

## 💻 Comandos utilizados

```bash
# Ajustar permissão da chave privada
chmod 400 chavedesafioec2.pem

# Conectar via SSH à instância
ssh -i "chavedesafioec2.pem" ec2-user@ec2-3-88-231-210.compute-1.amazonaws.com

# Validar usuário conectado
whoami
# → ec2-user

# Verificar versão do sistema operacional
cat /etc/os-release
# → Amazon Linux 2023.12

# Verificar espaço em disco (confirma o volume EBS de 8 GiB)
df -h
# → /dev/nvme0n1p1   8.0G   1.6G   6.4G   21%   /
```

---

## 🔁 Resultado do experimento: IP dinâmico

Um dos pontos mais didáticos da prática foi comparar o IP público **antes e depois** de parar e reiniciar a instância:

| Momento | Endereço IPv4 público |
|---|---|
| Instância lançada (1ª vez) | `3.88.231.210` |
| Após **Parar** → **Iniciar** novamente | `54.144.200.170` |

**Conclusão prática:** o IP público padrão atribuído pela AWS é **dinâmico** — ele muda a cada novo ciclo de start/stop. Isso é aceitável para testes, mas inviável para produção (ex: um domínio apontando para o servidor). Para resolver isso, a AWS oferece o **Elastic IP**, um endereço estático que permanece o mesmo independentemente de quantas vezes a instância for reiniciada.

---

## 🚧 Desafios encontrados

- Ao tentar parar a instância pela primeira vez, cliquei sem querer em **"Encerrar (excluir) instância"** em vez de **"Parar instância"** — a AWS abriu uma tela de confirmação avisando que a ação era destrutiva e apagaria o volume EBS. Consegui identificar a tela de alerta e cancelar antes de confirmar, evitando perder a instância.
- Pequena confusão de tradução: o status **"Interrompido"** no console em português é o mesmo que **"Stopped"** — não indica erro, é o estado esperado após parar a instância.
- Ao colar o comando SSH no Git Bash, a primeira tentativa retornou `command not found` por causa de um caractere de escape de terminal (`\E[200~`) — resolvido rodando o comando novamente direto no prompt.

---

## ✅ Boas práticas aprendidas

- **Nunca usar a conta root da AWS no dia a dia** — o ideal é criar um usuário IAM dedicado, com MFA ativado no root e guardado à parte.
- Restringir a porta SSH (22) ao **próprio IP**, nunca liberar para `0.0.0.0/0`.
- Diferenciar claramente **Parar (Stop)** de **Encerrar (Terminate)** antes de confirmar qualquer ação no console — a segunda é irreversível.
- Nunca versionar o arquivo de chave privada (`.pem`) em repositórios públicos.
- Usar Elastic IP quando o caso de uso exigir um endereço fixo (ex: apontar um domínio).
- Acompanhar o painel de créditos do Free Tier para evitar cobranças inesperadas.

---

## 🖼️ Imagens

Capturas de tela do processo estão disponíveis na pasta [`/images`](./images), incluindo:
- Console de lançamento da instância (nome, AMI, tipo de instância)
- Configuração do Security Group (SSH restrito ao IP local)
- Terminal com conexão SSH ativa e comandos de validação
- Tela de confirmação de execução da instância
- Instância nos estados "Executando" e "Interrompido"

---

## 📚 Referências

- [Documentação oficial da AWS — Amazon EC2](https://docs.aws.amazon.com/pt_br/AWSEC2/latest/UserGuide/concepts.html)
- [Gerenciando instâncias EC2 da Amazon — AWS Toolkit](https://docs.aws.amazon.com/pt_br/toolkit-for-visual-studio/latest/user-guide/tkv-ec2-ami.html)
- [GitHub Quick Start — DIO](https://github.com/digitalinnovationone/github-quickstart)
- [GitHub Markdown Guide](https://docs.github.com/pt/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
- [Formação GitHub Certification (GitBook)](https://aline-antunes.gitbook.io/formacao-fundamentos-github)

---

## 👤 Autor

**Willian Ferraz Kuhn**
Desafio de projeto — Digital Innovation One (DIO)
