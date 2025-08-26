# 📝 Relatório do Laboratório 2 - Chamadas de Sistema

---

## 1️⃣ Exercício 1a - Observação printf() vs 1b - write()

### 💻 Comandos executados:
```bash
strace -e write ./ex1a_printf
strace -e write ./ex1b_write
```

### 🔍 Análise

**1. Quantas syscalls write() cada programa gerou?**
- ex1a_printf: 9 syscalls
- ex1b_write: 7 syscalls

**2. Por que há diferença entre os dois métodos? Consulte o docs/printf_vs_write.md**

```
A diferença está no buffering: printf() guarda as mensagens em um buffer e só envia para o sistema quando necessário, podendo gerar mais ou menos syscalls dependendo da situação, enquanto write() envia direto para o sistema operacional com comportamento previsível onde cada chamada gera exatamente 1 syscall. Portanto o printf() otimiza performance agrupando escritas, enquanto write() dá controle total sobre quando os dados são enviados.
```

**3. Qual método é mais previsível? Por quê você acha isso?**

```
Eu acredito que o método write() seja mais previsível pois ele estabelece uma relação direta e constante onde cada chamada write() no código resulta em exatamente uma syscall para o kernel, enquanto o printf() tem comportamento variável devido ao buffer interno.
```

---

## 2️⃣ Exercício 2 - Leitura de Arquivo

### 📊 Resultados da execução:
- File descriptor: 3
- Bytes lidos: 127

### 🔧 Comando strace:
```bash
strace -e openat,read,close ./ex2_leitura
```

### 🔍 Análise

**1. Qual file descriptor foi usado? Por que não começou em 0, 1 ou 2?**

```
O file descriptor utilizado foi o 3, e ele nao começou em 0, 1 ou 2 porque esses flies descriptors estão reservados para as seguintes funções respectivamente: stdin, stdout, stderr. Portanto quando um programa abre um novo arquivo, o sistema operacional atribui o menor número disponível, que no caso é o 3.
```

**2. Como você sabe que o arquivo foi lido completamente?**

```
Eu sei que o arquivo foi lido completamente através da comparação do número de bytes lidos com o tamanho do arquivo, assim sabendo se ele foi lido completamente.
```

**3. Por que verificar retorno de cada syscall?**

```
Verificar o retorno de cada syscall é fundamental porque syscalls podem falhar por diversos motivos, e falhas não tratadas causam comportamentos imprevisíveis.
```

---

## 3️⃣ Exercício 3 - Contador com Loop

### 📋 Resultados (BUFFER_SIZE = 64):
- Linhas: 25 (esperado: 25)
- Caracteres: 1300
- Chamadas read(): 21
- Tempo: 0.000083 segundos

### 🧪 Experimentos com buffer:

| Buffer Size | Chamadas read() | Tempo (s) |
|-------------|-----------------|-----------|
| 16          |    82             |    0.000195       |
| 64          |    21             |     0.000083      |
| 256         |     6            |     0.000070      |
| 1024        |       2          |       0.000063    |

### 🔍 Análise

**1. Como o tamanho do buffer afeta o número de syscalls?**

```
O tamanho do buffer afeta no número das syscalls devido ao sue tamanho, quanto maior o buffer, menos syscalls ele realizará, consequentemente mais rápido ele será. 
```

**2. Todas as chamadas read() retornaram BUFFER_SIZE bytes? Discorra brevemente sobre**

```
Não, analisando o strace vemos que as primeiras 20 chamadas retornaram 64 bytes cada, mas a chamada 21 retornou apenas 20 bytes porque chegou ao fim do arquivo, e a chamada 22 retornou 0 bytes indicando o final. Então após 20 leituras de 64 bytes, sobraram apenas 20 bytes para a última leitura, demonstrando que read() não garante sempre ler o número completo solicitado.
```

**3. Qual é a relação entre syscalls e performance?**

```
Syscalls são lentas porque o programa precisa "pedir permissão" ao sistema operacional para fazer qualquer coisa, e isso demora. É como se a cada operação você tivesse que parar, ir até o balcão de atendimento, esperar na fila, fazer o pedido e voltar - muito mais lento que fazer as coisas direto na sua mesa. Por isso é melhor juntar várias tarefas e ir ao balcão uma vez só do que ir várias vezes para pequenas coisas. No nosso exemplo, fazer 21 chamadas read() pequenas é mais lento que fazer 2 chamadas read() grandes, mesmo lendo a mesma quantidade total de dados. Portanto quanto menos syscalls o programa realizar, mais rápido ele será.
```

---

## 4️⃣ Exercício 4 - Cópia de Arquivo

### 📈 Resultados:
- Bytes copiados: 1364
- Operações: 6
- Tempo: 0.000179 segundos
- Throughput: 7441.52 KB/s

### ✅ Verificação:
```bash
diff dados/origem.txt dados/destino.txt
```
Resultado: [X] Idênticos [ ] Diferentes

### 🔍 Análise

**1. Por que devemos verificar que bytes_escritos == bytes_lidos?**

```
Porque write() pode escrever menos bytes que o solicitado devido a problemas como disco cheio ou interrupções do sistema, e se isso acontecer os dados restantes serão perdidos, corrompendo o arquivo copiado. Verificar se bytes_escritos == bytes_lidos garante que todos os dados lidos foram efetivamente escritos no destino.
```

**2. Que flags são essenciais no open() do destino?**

```
As flags essenciais são O_WRONLY | O_CREAT | O_TRUNC porque O_WRONLY abre o arquivo apenas para escrita, O_CREAT cria o arquivo se ele não existir, e O_TRUNC limpa o conteúdo do arquivo se ele já existir (evitando misturar dados novos com antigos). Sem essas três flags, o programa não conseguiria criar um arquivo novo ou sobrescrever corretamente um arquivo existente.
```

**3. O número de reads e writes é igual? Por quê?**

```
Sim, o número de reads e writes deve ser igual porque o algoritmo de cópia funciona em pares: para cada read() que lê dados do arquivo origem, há exatamente um write() correspondente que escreve esses mesmos dados no arquivo destino.
```

**4. Como você saberia se o disco ficou cheio?**

```
O write() falharia e retornaria -1, fazendo o programa parar na verificação if (bytes_escritos != bytes_lidos) e mostrar uma mensagem de erro como "No space left on device". 
```

**5. O que acontece se esquecer de fechar os arquivos?**

```
O sistema operacional tem um limite de quantos arquivos cada programa pode ter aberto ao mesmo tempo, então se você esquecer de fechar, eventualmente o programa não conseguirá abrir mais arquivos e dará erro.
```

---

## 🎯 Análise Geral

### 📖 Conceitos Fundamentais

**1. Como as syscalls demonstram a transição usuário → kernel?**

```
As syscalls demonstram essa transição porque quando o programa chama funções como open(), read() ou write(), ele precisa sair do "modo usuário" (onde roda normalmente) e entrar no "modo kernel" (onde o sistema operacional controla o hardware). É como sair da sua casa e ir ao banco - você não pode mexer diretamente no cofre do banco, então precisa pedir para o funcionário (kernel) fazer isso por você. O strace mostra exatamente essas "idas ao banco", revelando cada momento que o programa parou de executar código próprio e pediu ajuda ao sistema operacional para acessar arquivos, tela ou outros recursos que só o kernel pode controlar.
```

**2. Qual é o seu entendimento sobre a importância dos file descriptors?**

```
File descriptors são identificadores únicos que funcionam como "números de protocolo" para o sistema operacional controlar quais arquivos cada programa está usando. Eles são essenciais porque permitem que o kernel mantenha controle total sobre os recursos
```

**3. Discorra sobre a relação entre o tamanho do buffer e performance:**

```
O tamanho do buffer tem uma relação direta com performance devido ao número de syscalls necessárias: buffers maiores reduzem drasticamente o número de chamadas ao sistema porque cada read() ou write() traz/envia mais dados de uma vez, enquanto buffers pequenos forçam muitas syscalls custosas.
```

### ⚡ Comparação de Performance

```bash
# Teste seu programa vs cp do sistema
time ./ex4_copia
time cp dados/origem.txt dados/destino_cp.txt
```

**Qual foi mais rápido?** O mais rápido foi time ./ex4_copia

**Por que você acha que foi mais rápido?**

```
 porque é um programa muito simples e específico que faz apenas uma coisa: copiar um arquivo pequeno usando syscalls diretas.
```

---

## 📤 Entrega
Certifique-se de ter:
- [X] Todos os códigos com TODOs completados
- [X] Traces salvos em `traces/`
- [X] Este relatório preenchido como `RELATORIO.md`

```bash
strace -e write -o traces/ex1a_trace.txt ./ex1a_printf
strace -e write -o traces/ex1b_trace.txt ./ex1b_write
strace -o traces/ex2_trace.txt ./ex2_leitura
strace -c -o traces/ex3_stats.txt ./ex3_contador
strace -o traces/ex4_trace.txt ./ex4_copia
```
# Bom trabalho!
