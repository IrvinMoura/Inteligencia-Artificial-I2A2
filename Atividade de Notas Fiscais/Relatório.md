#### **1. Objetivo do Projeto**

O projeto teve como objetivo principal o desenvolvimento de um agente de Inteligência Artificial capaz de interpretar perguntas de usuários em linguagem natural (português), consultar uma base de dados de notas fiscais e retornar respostas precisas e coesas. A solução foi implementada utilizando a plataforma de automação de workflows n8n, integrada a um banco de dados PostgreSQL e ao Large Language Model (LLM) Google Gemini.

#### **2. Arquitetura da Solução**

A solução foi arquitetada em um fluxo sequencial de processamento de dados, orquestrado pelo n8n. A arquitetura se destaca pela **separação de responsabilidades**, utilizando dois agentes de IA distintos para tarefas diferentes, o que garante maior precisão e flexibilidade.

O fluxo de dados segue o seguinte caminho:

1.  **Frontend (Chat HTML):** O usuário digita a pergunta.
2.  **n8n Webhook (Entrada):** Recebe a pergunta do usuário.
3.  **Agente de IA 1 (Gerador de SQL):** Converte a pergunta em uma consulta PostgreSQL.
4.  **Banco de Dados (PostgreSQL):** Executa a consulta e retorna os dados brutos.
5.  **Agente de IA 2 (Formatador de Resposta):** Recebe os dados brutos e a pergunta original para gerar uma resposta em linguagem natural.
6.  **n8n Respond to Webhook (Saída):** Envia a resposta final de volta para o Frontend.

#### **3. Detalhamento do Fluxo de Trabalho (Workflow)**

O workflow é composto por uma série de nós, cada um com uma função específica:

1.  **`Webhook`**: É o ponto de entrada do sistema. Ele fica ativo aguardando receber uma pergunta do usuário (via método POST) vinda da interface de chat.

2.  **`Edit Fields` (Preparação)**: Este nó recebe os dados do Webhook e isola a mensagem do usuário em um campo limpo chamado `ChatInput`. Isso organiza os dados para o próximo passo.

3.  **`Gera query SQL` (Agente de IA 1)**: Este é o primeiro "cérebro". Ele utiliza um modelo Google Gemini com um prompt detalhado que o instrui a agir como um especialista em PostgreSQL.
    * **Entrada:** A pergunta do usuário (`ChatInput`).
    * **Processo:** Analisa a pergunta e, seguindo as regras e o esquema do banco de dados definidos no prompt, gera uma query SQL válida e otimizada.
    * **Saída:** Um texto puro contendo a consulta SQL.

4.  **`Postgres`**: Este nó atua como o conector do banco de dados.
    * **Entrada:** A query SQL gerada pelo nó anterior.
    * **Processo:** Executa a consulta na base de dados que contém as tabelas `nfs_cabecalho` e `nfs_itens`.
    * **Saída:** Um array de objetos JSON, onde cada objeto representa uma linha do resultado da consulta.

5.  **`Basic LLM Chain` (Agente de IA 2)**: Este é o segundo "cérebro", responsável pela comunicação com o usuário.
    * **Entrada:** O resultado em JSON do nó Postgres e a pergunta original do usuário.
    * **Processo:** Utilizando um segundo prompt focado em comunicação, ele analisa os dados brutos no contexto da pergunta original e gera uma frase coesa, amigável e formatada.
    * **Saída:** Um texto puro contendo a resposta final para o usuário.

6.  **`Edit Fields1` (Finalização)**: Isola o texto da resposta final em um campo limpo chamado `response`.

7.  **`Respond to Webhook`**: É o ponto de saída. Ele envia a resposta final de volta para a interface de chat que iniciou a requisição, garantindo que o usuário veja o resultado.

#### **4. Inteligência e Lógica Implementada**

O sucesso do projeto reside na lógica complexa e nas instruções detalhadas fornecidas aos agentes de IA através da engenharia de prompt. As principais inteligências são:

* **Compreensão de Linguagem Natural:** O primeiro LLM interpreta sinônimos ("NF", "nota", "nota fiscal"), intenções (contar, somar, listar os maiores) e entidades (nomes de empresas, produtos).
* **Tratamento de Tipos de Dados em SQL:** Foi implementada uma regra crucial que ensina o LLM a converter colunas de texto (VARCHAR) que contêm valores monetários em tipos numéricos usando a função `CAST(REPLACE(REPLACE(...)))`, permitindo a execução de cálculos matemáticos.
* **Lógica de Ranking e Limites:** O prompt foi refinado para diferenciar perguntas que pedem o "maior" ou "melhor" (`LIMIT 1`) daquelas que pedem uma lista, como "os 5 primeiros" (`LIMIT 5`), garantindo que a consulta SQL retorne o número correto de resultados.
* **Conexão de Dados (`JOIN`):** O agente foi instruído a criar consultas com `INNER JOIN` sempre que uma pergunta exigir a combinação de informações das tabelas de cabeçalho e de itens (ex: "Qual cliente comprou o produto X?").
* **Processamento de Listas:** O segundo LLM foi configurado (com `Execute Once` e a expressão `$items()`) para receber e processar listas inteiras de resultados de uma só vez, permitindo-lhe gerar resumos coesos de múltiplos itens (ex: "Os 5 fornecedores são: A, B, C...").
* **Formatação de Saída:** O segundo LLM possui regras para formatar números como valores monetários (R$ XX.XXX,XX) ou inteiros com separadores de milhar, tornando a resposta final clara e profissional.

#### **5. Conclusão**

O workflow desenvolvido atende com sucesso ao objetivo proposto. A solução implementada é robusta, capaz de lidar com uma variedade de perguntas complexas, e demonstra uma arquitetura de IA eficaz baseada na separação de tarefas. O agente não apenas executa consultas, mas também gerencia a lógica de tratamento de dados e a formatação da comunicação, entregando uma experiência de usuário funcional e intuitiva.