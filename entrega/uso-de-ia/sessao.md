as vezes que eu mandei, estão marcadas com #.

# eu como faco isso? passo a passo

# nao resolvee, me ensina

Vou abrir o zip para ver o que o desafio pede.

Esse desafio é de depuração. O CLI gera um relatório JSON de reservas de hotel com números errados, e você precisa achar os bugs, corrigir, fazer o pytest passar e entregar no GitHub. Vou te dar o método e pistas de onde olhar, sem entregar as correções.

Passo 1: Prepare o ambiente
bash
cd onfly-desafio
python3.11 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
Passo 2: Veja tudo falhar
bash
pytest -q

Os testes falham no começo, e isso é esperado. Cada teste falho aponta para um defeito. Os arquivos são:

test_status_rules.py
test_nights.py
test_currency.py
test_top_hotels.py
test_regression_report.py

Rode um arquivo por vez (pytest tests/test_nights.py -q) e leia a mensagem assert X == Y. Ela mostra o valor obtido e o esperado.

Passo 3: Rode o CLI na amostra pequena
bash
PYTHONPATH=src python -m onfly_report \
  --input data/exemplo-validacao.csv \
  --rates data/taxas_cambio.csv \
  --output output/exemplo.json

Compare o resultado com a tabela “Amostra de validação” do README. São 12 linhas, então dá para conferir uma a uma na mão. Esse é o jeito mais rápido de ver qual número diverge.

Passo 4: Mapeie sintoma → arquivo

A pasta src/onfly_report/services/ tem um arquivo para cada regra de negócio. Isso ajuda, porque cada regra do README corresponde a um arquivo pequeno. Siga esta ordem de investigação, uma regra por vez:

Sintoma	Arquivo para ler	Pergunta para se fazer
Contagem de faturáveis ou valores incluem coisa demais	status_policy.py	O README diz que só CONFIRMED é faturável. A condição escrita é equivalente a isso? Pense no NO_SHOW.
Noites erradas, principalmente na virada de mês	stay_calculator.py	A fórmula é (checkout - checkin).days * rooms. O que o código usa no lugar de .days? Teste na cabeça com check-in 30/01 e check-out 02/02.
Valores em BRL levemente errados	currency_converter.py	Compare com a regra: amount * rate. Veja a operação usada e os tipos (float vs Decimal).
Ranking dos hotéis invertido	report_builder.py, em ranked_hotels	A regra pede total_brl decrescente e, no empate, hotel_id ascendente. Como se faz isso numa chave do sorted? Dica: valor negado, ou dois sorts.
Totais não batem no arquivo grande (reservas_hoteis.csv)	repositories/csv_reservation_repository.py	Esse arquivo tem 300 linhas de dados. Leia o for e se pergunte se ele lê todas.

O README fala de “ordem do processamento” e de “arredondar só na serialização”. Depois de achar os bugs acima, releia o report_builder.py com essas duas regras na mão e confira se algo viola alguma delas.

Passo 5: Corrija um bug por vez

O ciclo para cada defeito é:

Corrija o código.
Rode pytest -q e veja quantos testes passaram a mais.
Rode o CLI na amostra e compare com a tabela.
Faça um commit pequeno, por exemplo fix: cálculo de noites na virada de mês.

Os commits pequenos ajudam no README-ENTREGA.md, porque cada um é um defeito documentado.

Passo 6: Valide no dataset real
bash
PYTHONPATH=src python -m onfly_report \
  --input data/reservas_hoteis.csv \
  --rates data/taxas_cambio.csv \
  --output output/report.json

Confira:

total_reservations deve ser 300.
O dinheiro deve ser string com 2 casas.
top_hotels deve ter 5 itens.
Passo 7: Escreva a entrega

Crie entrega/README-ENTREGA.md com estas seções:

Defeitos corrigidos: uma linha de causa raiz por defeito (arquivo, o que estava errado, por quê).
Como investiguei: o que você rodou e em que ordem, como o passo 2 e o passo 3.
Como validar: os comandos do passo 3 e do passo 6.
Resultado do pytest: cole a saída final.

Se usou IA, como está fazendo agora, o README exige uma pasta entrega/uso-de-ia/ com:

sessao.md, que é a exportação desta conversa. No Claude Code isso é o /export; aqui, copie a conversa para um arquivo.
declaracao.md, com a ferramenta e o modelo usados, o que a IA fez e o que você fez. Seja honesto e específico, por exemplo: “a IA explicou o método e deu pistas, eu localizei e corrigi os bugs”.
Passo 8: Publique
bash
git init
git add .
git commit -m "Correções do desafio Onfly"
# crie um repositório PÚBLICO no GitHub, depois:
git remote add origin <url-do-repo>
git branch -M main
git push -u origin main

Envie o link pelo formulário.

Regras para não errar
Não altere nada em tests/ nem em data/reservas_hoteis.csv e data/taxas_cambio.csv.
Não mude os nomes dos argumentos do CLI nem o formato do JSON.
Para dinheiro, use Decimal, nunca float. Arredonde só no final, ao formatar.

Se travar em algum bug, me mande o trecho do código e a saída do teste que falha. Eu te dou uma dica mais direcionada, e você continua escrevendo a correção.

# da mais detalhes de como fazer

Cada bug abaixo tem um experimento que você roda para ver o erro acontecer. Quando enxergar o erro, a correção costuma ser uma linha.

Etapa 0: Como ler a saída do pytest
bash
pytest tests/test_nights.py -q

A saída tem esta forma:

E   assert 3 == 6        <- esquerda = o que seu código devolveu, direita = o esperado
tests/test_nights.py:12  <- a linha do teste que falhou

Abra o teste e veja quais valores ele passa para a função. Depois reproduza a chamada no terminal Python, como nos experimentos abaixo.

Para testar uma função isolada:

bash
cd onfly-desafio
PYTHONPATH=src python
python
>>> from datetime import date
>>> from onfly_report.services.stay_calculator import calculate_room_nights
>>> calculate_room_nights(date(2025,1,30), date(2025,2,2), 1)
Bug 1: Status (services/status_policy.py)

Experimento:

python
>>> from onfly_report.services.status_policy import is_billable
>>> for s in ["CONFIRMED", "CANCELLED", "NO_SHOW"]:
...     print(s, is_billable(s))

O que observar: o README diz que só um status é faturável. Quantos True apareceram?

Raciocínio: a função usa uma negação (!=) de um único status. Negar um valor deixa passar todos os outros, inclusive os que você não quer. Pergunte-se qual comparação positiva descreve a regra “só CONFIRMED”.

Bug 2: Noites (services/stay_calculator.py)

Experimento 1: veja como o Python subtrai datas

python
>>> from datetime import date
>>> d = date(2025,2,2) - date(2025,1,30)
>>> d
>>> type(d)
>>> d.days

Experimento 2: veja o que o código atual usa

python
>>> date(2025,2,2).day, date(2025,1,30).day

O que observar: o dia do mês (.day) é um número de 1 a 31. Compare a diferença entre .day e .days para check-in 30/01 e check-out 02/02. Uma delas dá negativo. Qual?

Raciocínio: .day ignora o mês. O README pede a diferença entre duas datas completas, e a fórmula já está escrita nele.

Bug 3: Câmbio (services/currency_converter.py)

Este bug tem dois problemas juntos, e os dois estão no comentário do README.

Experimento 1: a operação

python
>>> amount, rate = 200, 5.42
>>> amount * rate      # o que a regra diz
>>> amount / rate      # o que o código faz

Veja qual dos dois resultados faz sentido para US$ 200 em reais. Se o hotel custa 200 dólares e o dólar vale mais de 1 real, o valor em BRL deve ser maior ou menor que 200?

Experimento 2: float vs Decimal

python
>>> 0.1 + 0.2
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")

O float guarda aproximações binárias, e por isso o README proíbe float com dinheiro. A função atual faz float(amount), depois divide, depois str(...). Cada etapa perde precisão.

Raciocínio: amount e rate já chegam como Decimal. A multiplicação de Decimal por Decimal é exata, então nenhuma conversão é necessária.

Bug 4: Ranking (services/report_builder.py, variável ranked_hotels)

Experimento: entenda o que o sorted faz com tuplas

python
>>> dados = [("H01", 100), ("H02", 300), ("H03", 300)]
>>> sorted(dados, key=lambda x: (x[1], x[0]))

O Python ordena crescente por padrão e compara os elementos da tupla em ordem. A regra do README pede:

valor decrescente;
empate: hotel_id crescente.

Há uma regra, mas ela vale para números, não para strings: se você negar o valor (-x[1]), a ordem do número inverte, e o hotel_id continua crescente. Teste no exemplo acima. Os dois 300 devem aparecer na ordem H02, H03, antes de H01.

Outra opção é reverse=True, mas ela inverte tudo, inclusive o hotel_id. Verifique se isso quebra o desempate.

Dica: o amount dos hotéis é Decimal. Dá para negar um Decimal sem problema.

Bug 5: Limite de linhas (repositories/csv_reservation_repository.py)

Experimento: o arquivo tem quantas linhas de dados?

bash
wc -l data/reservas_hoteis.csv

Subtraia 1 do resultado (o cabeçalho). Agora rode o CLI e veja o total_reservations impresso. Os dois números são iguais?

Olhe o loop for values in reader:. Há um if dentro dele que faz algo quando len(rows) == 200. Releia o que ele faz. É uma regra de negócio ou é uma trava que ninguém pediu?

Etapa 6: Procure bugs que não têm teste

Os testes cobrem partes do contrato, mas a avaliação também olha o README. Percorra a seção “Exatidão” e confira:

O arredondamento acontece só na hora de formatar? Procure em report_builder.py por quantize e veja onde ele é chamado.
O average_nightly_rate_brl usa o total exato ou um total já arredondado?
O by_month usa o mês do check-in?

Se encontrar algo extra, melhor ainda: vira um defeito a mais no README-ENTREGA.

Etapa 7: Teste que você mesmo escreve

O README valoriza testes novos. Crie tests/test_meu_caso.py (sem tocar nos existentes). Um bom candidato é a virada de ano:

python
from datetime import date
from onfly_report.services.stay_calculator import calculate_room_nights

def test_virada_de_ano():
    assert calculate_room_nights(date(2024,12,30), date(2025,1,2), 1) == 3

Faça um teste por bug corrigido. Isso mostra que você entendeu o problema, não só o conserto.

Etapa 8: Confirmação final
bash
pytest -q                      # tudo verde
ruff check src tests           # opcional: estilo

Depois rode o CLI na amostra de 12 linhas e confira cada número da tabela do README:

| Campo | Esper

# fala como eu começo desde o inicio, passo a passo

Planning a beginner-friendly setup walkthrough.

Começando do zero, sem pular nada. Faça uma etapa por vez e só passe para a próxima quando a anterior funcionar.

Etapa 1: Descompacte o zip

Clique com o botão direito no arquivo .zip e escolha “Extrair tudo” (Windows) ou dê dois cliques (Mac). Você terá uma pasta onfly-desafio. Mova essa pasta para um lugar fácil, como a Área de Trabalho ou Documentos.

Etapa 2: Veja se o Python está instalado

Abra o terminal:

Windows: tecle Win, digite cmd e Enter. (O PowerShell também serve.)
Mac: Cmd + Espaço, digite Terminal e Enter.

Digite:

bash
python --version

Se não funcionar, tente python3 --version. O desafio pede Python 3.11 ou mais novo. Se aparecer um número menor ou der erro, instale em python.org/downloads. No Windows, marque a caixa “Add Python to PATH” durante a instalação. Depois feche e abra o terminal de novo.

Etapa 3: Entre na pasta do projeto pelo terminal
bash
cd Desktop/onfly-desafio

Ajuste o caminho para onde você colocou a pasta. Se não souber, digite cd  (com espaço) e arraste a pasta para dentro da janela do terminal. Confira se está no lugar certo:

bash
ls        # Mac/Linux
dir       # Windows

Você deve ver README.md, src, tests, data e os outros arquivos.

Etapa 4: Instale um editor de código

Se ainda não tem, instale o VS Code (code.visualstudio.com). Abra o VS Code, vá em File → Open Folder e escolha onfly-desafio. À esquerda aparece a árvore de arquivos. É aqui que você vai ler e editar o código.

Etapa 5: Crie o ambiente virtual

Ele isola as bibliotecas deste projeto das do resto do computador. No terminal, dentro da pasta:

bash
python -m venv .venv

Ative:

bash
# Mac/Linux
source .venv/bin/activate

# Windows (cmd)
.venv\Scripts\activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

Quando funcionar, aparece (.venv) no começo da linha do terminal.

Etapa 6: Instale o pytest
bash
pip install -r requirements-dev.txt
Etapa 7: Rode os testes e veja eles falharem
bash
pytest -q

No começo vários testes falham, e isso é esperado. No final aparece algo como 5 failed. Esse número é o seu ponto de partida. A meta é chegar a tudo passando.

Etapa 8: Rode o programa na amostra pequena

Mac/Linux:

bash
PYTHONPATH=src python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

Windows (cmd):

bash
set PYTHONPATH=src
python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

Windows (PowerShell):

bash
$env:PYTHONPATH="src"
python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

Depois abra output/exemplo.json no VS Code e compare com a tabela “Amostra de validação” do README.md. Os números estão errados, e achar o motivo é o desafio.

Etapa 9: Corrija os bugs, um por vez

A partir daqui vale o guia que mandei antes. A ordem recomendada é esta:

services/status_policy.py
services/stay_calculator.py
services/currency_converter.py
services/report_builder.py (a ordenação do ranking)
repositories/csv_reservation_repository.py

A cada correção, salve o arquivo (Ctrl+S), rode pytest -q e veja se o número de falhas diminuiu.

Etapa 10: Escreva a entrega

Crie o arquivo entrega/README-ENTREGA.md com os defeitos corrigidos, como você investigou, como validar e o resultado final do pytest. Se usou IA, crie também entrega/uso-de-ia/declaracao.md e sessao.md, como explicado antes.

Etapa 11: Publique no GitHub
Crie uma conta em github.com, se não tiver.
Clique em New repository, dê um nome, marque Public e crie (sem README).
No terminal, dentro da pasta do projeto:
bash
git init
git add .
git commit -m "Correções do desafio Onfly"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/NOME_DO_REPO.git
git push -u origin main
Abra o link do repositório no navegador e confira que entrega/ e src/ aparecem.
Mande o link pelo formulário da Onfly.

Comece pelas etapas 1 a 8 e me diga onde travou, ou cole aqui o que apareceu no pytest -q. Daí a gente segue para o primeiro bug.

# comecei a etapa 5 e nao entendi bem

A etapa 5 tem duas partes: criar o ambiente virtual e ativá-lo.

O que é o ambiente virtual

É uma pasta (.venv) dentro do projeto com uma cópia isolada do Python. Tudo o que você instalar com pip fica ali dentro e não bagunça o resto do computador. Não é obrigatório, mas é o padrão e é o que o README pede.

Parte A: Criar

No terminal, dentro da pasta onfly-desafio, digite:

bash
python -m venv .venv

Aperte Enter e espere alguns segundos. Não aparece mensagem de sucesso, e isso é normal. Para conferir, olhe a árvore de arquivos no VS Code: deve ter aparecido uma pasta .venv.

Se deu erro python não é reconhecido ou command not found, tente python3 -m venv .venv (Mac/Linux) ou py -m venv .venv (Windows).

Parte B: Ativar

Criar não basta, você precisa ligar o ambiente. Escolha o comando do seu sistema:

Mac/Linux:
bash
source .venv/bin/activate
Windows com cmd (tela preta):
bash
.venv\Scripts\activate
Windows com PowerShell (tela azul):
bash
.venv\Scripts\Activate.ps1

Como saber que funcionou: aparece (.venv) no começo da linha do terminal, assim:

(.venv) C:\Users\voce\Desktop\onfly-desafio>
Problemas comuns
PowerShell dá erro de “execução de scripts desabilitada”: rode isto uma vez e tente ativar de novo:
bash
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned

Ou use o cmd, que não tem esse bloqueio.

Você não está dentro da pasta certa: o terminal precisa estar em onfly-desafio. Digite dir (Windows) ou ls (Mac/Linux) e confira se aparece README.md.
Fechou o terminal: o ambiente desativa. Abra de novo, entre na pasta e rode só a Parte B (ativar). Não precisa criar outra vez.

Me diga qual sistema você usa (Windows ou Mac) e o que apareceu na tela, se algo deu errado. Copie a mensagem de erro exata, que eu te digo o que fazer.

# onde eu digito python -m venv .venv

No terminal, não em um arquivo. Para abrir o terminal dentro do projeto:

Opção mais fácil: terminal do VS Code
Abra o VS Code com a pasta onfly-desafio aberta (File → Open Folder).
No menu de cima, clique em Terminal → New Terminal. Atalho: Ctrl + ` (a tecla da crase, abaixo do Esc).
Abre uma janela embaixo do VS Code. A linha mostra algo como C:\...\onfly-desafio>. O final do caminho precisa ser onfly-desafio, e isso significa que você já está na pasta certa.
Clique nessa janela, digite o comando e aperte Enter:
bash
python -m venv .venv
Opção alternativa: terminal do sistema
Windows: abra a pasta onfly-desafio no Explorador de Arquivos, clique na barra de endereço no topo, apague o texto, digite cmd e aperte Enter. Abre um terminal já dentro da pasta.
Mac: abra o Terminal, digite cd  (com espaço no final), arraste a pasta onfly-desafio para dentro da janela e aperte Enter.

Depois digite o comando lá.

O que deve acontecer

Nada aparece na tela por alguns segundos, e depois o cursor volta. Isso quer dizer que deu certo. Na lista de arquivos à esquerda do VS Code deve aparecer uma pasta nova chamada .venv.

Aí passe para a Parte B (ativar), usando o comando do seu sistema que mandei antes.

Se o VS Code mostrar um pop-up perguntando “Do you want to select it for the workspace folder?” depois de criar, clique em Yes.

.venv\Scripts\activate : O arquivo C:\Users\usuario\Desktop\onfly-desafio\.venv\Scripts\Activate.ps1 não pode ser carregado porque a execução de scripts foi desabilitada 
neste sistema. Para obter mais informações, consulte about_Execution_Policies em https://go.microsoft.com/fwlink/?LinkID=135170.
No linha:1 caractere:1
+ .venv\Scripts\activate
+ ~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ErrodeSegurança: (:) [], PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess 

oq faco

Esse é o erro do PowerShell que mencionei. O Windows bloqueia scripts por padrão, e o comando de ativar é um script. A pasta .venv foi criada normalmente, só falta liberar a execução.

Opção 1: Liberar o PowerShell (recomendada)

No mesmo terminal, digite:

powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned

Ele pergunta se você quer mudar a política. Digite S (ou Y se estiver em inglês) e aperte Enter. Isso vale só para o seu usuário e não afeta o resto do sistema.

Depois ative de novo:

powershell
.venv\Scripts\activate

Deve aparecer (.venv) no começo da linha.

Opção 2: Usar o cmd em vez do PowerShell

Se não quiser mexer na política:

No terminal do VS Code, clique na seta ao lado do + (canto direito do painel do terminal).
Escolha Command Prompt.
Digite:
bash
.venv\Scripts\activate

O cmd não tem esse bloqueio.

Como saber que funcionou

A linha do terminal passa a começar com (.venv):

(.venv) PS C:\Users\usuario\Desktop\onfly-desafio>

Daí siga para a etapa 6:

bash
pip install -r requirements-dev.txt

Se aparecer outro erro, copie a mensagem aqui.

passo 8 deu isso

python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

C:\Users\usuario\Desktop\onfly-desafio\.venv\Scripts\python.exe: No module named onfly_report

Esse erro é porque faltou o PYTHONPATH=src. O código está dentro da pasta src/, e o Python não procura lá sozinho. Esse prefixo é para Mac/Linux. No PowerShell você define a variável em uma linha separada antes.

Faça assim

No mesmo terminal (PowerShell), rode primeiro:

powershell
$env:PYTHONPATH="src"

Não aparece nada, e isso é normal. Depois rode o comando de novo:

powershell
python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

Deve imprimir uma linha parecida com:

Reservas: 12 | Confirmadas: ... | Noites: ... | Total BRL: ...

Os números vão estar errados, e esse é o objetivo do desafio. Depois abra output/exemplo.json no VS Code.

Detalhes importantes
A variável $env:PYTHONPATH vale só para esse terminal. Se fechar e abrir outro, rode a linha de novo.
Você precisa estar na pasta onfly-desafio (o final do caminho na linha do terminal).
Se estiver no cmd em vez do PowerShell, a linha é set PYTHONPATH=src.

O pytest não precisa disso, porque o pyproject.toml já configura o caminho.

Quando rodar, cole aqui a linha que ele imprimiu. Comparamos com a tabela do README para ver quais números divergem, e isso mostra por qual bug começar.

oq é um cmd e oq é um powershell

Os dois são programas do Windows em que você digita comandos, o terminal. Fazem quase a mesma coisa, mas são diferentes por dentro.

cmd (Prompt de Comando)
É o mais antigo, com tela preta.
A linha começa assim: C:\Users\usuario\Desktop\onfly-desafio>
Para definir a variável: set PYTHONPATH=src
Para ativar o ambiente: .venv\Scripts\activate
Não tem o bloqueio de scripts.
PowerShell
É o mais novo e mais poderoso, geralmente com tela azul.
A linha começa com PS: PS C:\Users\usuario\Desktop\onfly-desafio>
Para definir a variável: $env:PYTHONPATH="src"
Tem o bloqueio de scripts, que você já viu.
Como saber qual você está usando

Olhe o começo da linha onde você digita:

Começa com PS → é PowerShell.
Começa só com C:\...> → é cmd.

O terminal do VS Code costuma abrir em PowerShell. Pelo erro que você mandou, você está no PowerShell, e por isso o comando do passo 8 é o $env:PYTHONPATH="src".

Qual usar

Tanto faz, mas escolha um e fique nele durante o desafio, para não misturar os comandos. Se o PowerShell estiver te dando dor de cabeça, troque para o cmd no VS Code: clique na seta ao lado do + no painel do terminal e escolha Command Prompt. Depois você precisa ativar o ambiente de novo (.venv\Scripts\activate) e usar set PYTHONPATH=src.

como eu corrijo os bugs? etapa 9

Corrigir um bug é sempre o mesmo ciclo. Vamos fazer o primeiro juntos e depois você repete nos outros.

O ciclo (vale para todos os bugs)
Abrir o arquivo no VS Code (clique nele na lista à esquerda).
Ler a função e entender o que ela faz hoje.
Comparar com a regra do README.
Editar o código.
Salvar (Ctrl+S).
Rodar pytest -q e ver se o número de falhas caiu.
Bug 1: faça agora

Passo 1. No VS Code, abra: src → onfly_report → services → status_policy.py. O arquivo é pequeno:

python
def is_billable(status: str) -> bool:
    """Indica se a reserva entra nos totais financeiros e de noites."""
    return status != "CANCELLED"

Passo 2. Leia em voz alta. “Devolve verdadeiro se o status for diferente de CANCELLED.” Então CONFIRMED dá verdadeiro e CANCELLED dá falso. E NO_SHOW?

Passo 3. Compare com o README, seção Status: “Só CONFIRMED é faturável.”

Pense: “diferente de CANCELLED” é a mesma coisa que “igual a CONFIRMED”? Só seria se existissem apenas dois status, e existem três.

Passo 4. Edite. Você precisa trocar o != por uma comparação que diga “é exatamente CONFIRMED”. Qual é o operador de igualdade em Python? Pense nele e troque a linha do return. Não mude mais nada no arquivo.

Passo 5. Salve com Ctrl+S. Se o nome do arquivo na aba do editor tiver uma bolinha, ainda não salvou.

Passo 6. Teste. Rode só o teste de status:

powershell
pytest tests/test_status_rules.py -q

Se passou, rode pytest -q completo e compare o número de falhas com o de antes. Deve ter diminuído.

Depois do bug 1, siga esta ordem
Bug	Arquivo	Teste para rodar
2. Noites	services/stay_calculator.py	pytest tests/test_nights.py -q
3. Câmbio	services/currency_converter.py	pytest tests/test_currency.py -q
4. Ranking	services/report_builder.py	pytest tests/test_top_hotels.py -q
5. Limite de linhas	repositories/csv_reservation_repository.py	pytest tests/test_regression_report.py -q

Para cada um, faça os mesmos seis passos: abrir, ler em voz alta, comparar com o README, editar, salvar, testar. As pistas e os experimentos de cada bug estão na minha mensagem anterior de detalhes. Use os experimentos no terminal Python para ver o erro antes de editar.

Se o teste continuar falhando

Leia a mensagem do pytest, a linha que começa com E:

E   assert 6 == 3

O número da esquerda é o que seu código devolveu, e o da direita é o esperado. Isso diz se você está errando para mais ou para menos.

Faça o bug 1 e me conte o resultado do pytest tests/test_status_rules.py -q. Se não conseguir, cole aqui o código que você escreveu e a saída, e eu te digo o que está errado sem dar a resposta pronta.

_______________________________________________________________ test_status_rules_exclude_cancelled_and_no_show _______________________________________________________________

tmp_path = WindowsPath('C:/Users/usuario/AppData/Local/Temp/pytest-of-usuario/pytest-1/test_status_rules_exclude_canc0')

    def test_status_rules_exclude_cancelled_and_no_show(tmp_path):
        """Somente CONFIRMED entra nos totais financeiros."""
        report = run_cli(
            tmp_path,
            [
                {"reservation_id": "R-1", "status": "CONFIRMED", "amount": "100.00"},
                {"reservation_id": "R-2", "status": "CANCELLED", "amount": "50.00"},
                {"reservation_id": "R-3", "status": "NO_SHOW", "amount": "20.00"},
            ],
        )
        assert report["summary"]["total_reservations"] == 3
        assert report["summary"]["confirmed_reservations"] == 1
        assert report["summary"]["cancelled_reservations"] == 1
        assert report["summary"]["no_show_reservations"] == 1

      assert report["summary"]["total_amount_brl"] == "100.00"

E       AssertionError: assert '50.00' == '100.00'
E         
E         - 100.00
E         + 50.00

tests\test_status_rules.py:20: AssertionError
=========================================================================== short test summary info ===========================================================================
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show - AssertionError: assert '50.00' == '100.00'
1 failed in 0.34s normal?

Normal, e o resultado diz exatamente o que está errado. Olhe os dados do teste:

R-1 CONFIRMED vale 100.00
R-2 CANCELLED vale 50.00
R-3 NO_SHOW vale 20.00

O seu código somou 50.00, que é o valor da reserva CANCELLED. Ou seja, agora só a cancelada está entrando nos totais, o contrário do que a regra pede. Isso acontece quando o return ficou com a comparação invertida, por exemplo comparando com "CANCELLED" em vez de "CONFIRMED".

O que fazer

Abra src/onfly_report/services/status_policy.py e olhe a linha do return. Confira duas coisas:

Qual status está escrito entre aspas? Precisa ser o único faturável pelo README.
Qual operador você usou? Precisa ser o de igualdade, não o de diferença.

Corrija, salve com Ctrl+S e rode de novo:

powershell
pytest tests/test_status_rules.py -q

Deve aparecer 1 passed.

Se continuar falhando, cole aqui o conteúdo do arquivo status_policy.py e eu te digo qual detalhe está errado.

"""Política de status das reservas."""

from __future__ import annotations


def is_billable(status: str) -> bool:
    """Indica se a reserva entra nos totais financeiros e de noites."""
    return status == "CANCELLED"

Achei. O operador está certo (==), mas o status entre aspas está errado.

python
return status == "CANCELLED"

Isso diz “é faturável se for CANCELLED”, o contrário do que a regra pede. Releia a frase do README: “Só CONFIRMED é faturável.”

Troque a palavra entre aspas pelo status faturável. Aí a linha lê como “é faturável se o status for igual a esse”.

Salve com Ctrl+S e rode:

powershell
pytest tests/test_status_rules.py -q

Deve aparecer 1 passed. Depois rode pytest -q completo para ver quantas falhas restam, e me diga o número. O próximo é o bug das noites (stay_calculator.py).

_______________________________________________________________ test_status_rules_exclude_cancelled_and_no_show _______________________________________________________________

tmp_path = WindowsPath('C:/Users/usuario/AppData/Local/Temp/pytest-of-usuario/pytest-5/test_status_rules_exclude_canc0')

    def test_status_rules_exclude_cancelled_and_no_show(tmp_path):
        """Somente CONFIRMED entra nos totais financeiros."""
        report = run_cli(
            tmp_path,
            [
                {"reservation_id": "R-1", "status": "CONFIRMED", "amount": "100.00"},
                {"reservation_id": "R-2", "status": "CANCELLED", "amount": "50.00"},
                {"reservation_id": "R-3", "status": "NO_SHOW", "amount": "20.00"},
            ],
        )
        assert report["summary"]["total_reservations"] == 3
        assert report["summary"]["confirmed_reservations"] == 1
        assert report["summary"]["cancelled_reservations"] == 1
        assert report["summary"]["no_show_reservations"] == 1

      assert report["summary"]["total_amount_brl"] == "100.00"

E       AssertionError: assert '0.00' == '100.00'
E         
E         - 100.00
E         ? --
E         + 0.00

tests\test_status_rules.py:20: AssertionError
=========================================================================== short test summary info ===========================================================================
FAILED tests/test_status_rules.py::test_status_rules_exclude_cancelled_and_no_show - AssertionError: assert '0.00' == '100.00'
1 failed in 0.28s

Progresso: antes entrava só a cancelada (50.00), agora não entra nenhuma (0.00). Então o arquivo foi salvo e a lógica mudou. Mas a reserva CONFIRMED (100.00) continua sendo barrada, e isso significa que a palavra entre aspas não bate exatamente com o status do CSV.

Confira o texto entre aspas, letra por letra

O Python compara texto de forma literal. Estes erros fazem a comparação falhar:

Letras minúsculas: "confirmed" ou "Confirmed". Tem que ser tudo maiúsculo.
Erro de digitação: "CONFIRMD", "CONFIRMADO", "CONFIRMED" com a letra O no lugar do zero, etc.
Espaço dentro das aspas: "CONFIRMED ".
Status errado: "NO_SHOW", por exemplo.
Como testar sem adivinhar

No terminal (com o ambiente ativado e $env:PYTHONPATH="src" definido), rode:

powershell
python -c "from onfly_report.services.status_policy import is_billable; print(is_billable('CONFIRMED'), is_billable('CANCELLED'), is_billable('NO_SHOW'))"

O resultado certo é:

True False False

Se aparecer False False False, o texto entre aspas no seu arquivo não bate com CONFIRMED. Copie a palavra direto do README e cole, em vez de digitar.

Se continuar sem achar, cole aqui o conteúdo atual de status_policy.py e o resultado desse comando.

# QUAL PALAVRA DO README

A palavra é CONFIRMED. O README diz: “Só CONFIRMED é faturável.”

A linha do return fica assim:

python
return status == "CONFIRMED"

Salve com Ctrl+S e rode:

powershell
pytest tests/test_status_rules.py -q

Deve aparecer 1 passed. Se der outro resultado, cole aqui o que apareceu.

# 1 passed

Bug 1 corrigido.

Agora o bug 2, as noites. É o mesmo ciclo: abrir, ler, comparar, editar, salvar, testar.

Passo 1: Abra o arquivo

src/onfly_report/services/stay_calculator.py. O código atual é:

python
return (checkout.day - checkin.day) * rooms
Passo 2: Veja o erro acontecer

No terminal, com o ambiente ativo e $env:PYTHONPATH="src" definido, abra o Python:

powershell
python

Depois digite, linha por linha:

python
>>> from datetime import date
>>> a = date(2025, 1, 30)
>>> b = date(2025, 2, 2)
>>> b.day - a.day
>>> b - a
>>> (b - a).days

Compare os três resultados. De 30/01 a 02/02 são 3 noites. Qual dos três resultados dá 3? Para sair do Python, digite exit().

Passo 3: Compare com o README

A regra está na seção Noites:

room_nights = (checkout_date - checkin_date).days * rooms
Passo 4: Edite

O código atual usa .day, que é só o dia do mês e ignora o mês. A regra usa a subtração das datas inteiras, seguida de .days. Reescreva a linha do return seguindo a fórmula do README, com checkout e checkin no lugar dos nomes.

Passo 5: Salve e teste
powershell
pytest tests/test_nights.py -q

Quando passar, rode pytest -q completo e me diga quantas falhas restam.

Checking the sample against the README table.

Todos os testes passaram. Antes de escrever a entrega, vou conferir se esses cinco bugs bastam para bater com a tabela do README, ou se sobrou algum defeito sem teste. Faço isso numa cópia minha, sem mexer nos seus arquivos.

Com os cinco consertos, o resultado bate com a tabela do README (12 / 9 / 2 / 1, 28 noites, 7566.00, 270.21, meses e ranking iguais). Os cinco defeitos eram todos. Você está na reta final.

Etapa 1: Confira na sua máquina
powershell
python -m onfly_report --input data/exemplo-validacao.csv --rates data/taxas_cambio.csv --output output/exemplo.json

A linha impressa deve ser:

Reservas: 12 | Confirmadas: 9 | Noites: 28 | Total BRL: 7566.00

Depois rode o arquivo grande:

powershell
python -m onfly_report --input data/reservas_hoteis.csv --rates data/taxas_cambio.csv --output output/report.json

Deve começar com Reservas: 300. Se algum número divergir, me diga qual.

Etapa 2: Escreva entrega/README-ENTREGA.md

No VS Code, clique com o botão direito na pasta entrega → New File → README-ENTREGA.md. Use este modelo e preencha as partes em maiúsculas com as suas palavras:

markdown
