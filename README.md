# Atividade: Assistente de IA Generativa com Hugging Face e Gemini

**Disciplina:** Disruptive Architectures: IoT, IoB & Generative AI
**Curso:** Análise e Desenvolvimento de Sistemas
**Base:** Aula 05, "Explorando IA Generativa com Hugging Face e Google Gemini" (segunda parte: APIs, chat, Gradio e integração)

---

## Sumário

1. [Objetivo](#1-objetivo)
2. [O que o grupo vai construir](#2-o-que-o-grupo-vai-construir)
3. [Temas](#3-temas)
4. [Conceitos importantes antes de começar](#4-conceitos-importantes-antes-de-começar)
5. [Pré-requisitos](#5-pré-requisitos)
6. [Configuração passo a passo](#6-configuração-passo-a-passo)
7. [As etapas da atividade](#7-as-etapas-da-atividade)
8. [Entrega](#8-entrega)
9. [Erros comuns](#9-erros-comuns)
10. [Segurança dos tokens](#10-segurança-dos-tokens)
11. [Estrutura deste repositório](#11-estrutura-deste-repositório)
12. [Referências](#12-referências)

---

## 1. Objetivo

Construir, em grupo, um **assistente de IA** sobre um tema da disciplina, usando duas APIs de modelos de linguagem:

- **Hugging Face Inference API**, com o modelo `meta-llama/Llama-3.1-8B-Instruct`
- **Google Gemini API**, com o modelo `gemini-2.5-flash-lite`

O assistente começa simples e vai ganhando recursos a cada etapa, até virar uma interface web e (no bônus) uma API REST.

---

## 2. O que o grupo vai construir

```
Etapa 1  Assistente com personalidade   (role "system" na prática)
   |
Etapa 2  Hugging Face x Gemini          (mesmas perguntas, dois modelos)
   |
Etapa 3  Chat com memória                (lista de histórico)
   |
Etapa 4  Interface web com Gradio        (escolha de modelo na tela)
   |
Etapa 5  (Bônus) API com FastAPI         (o assistente vira um endpoint REST)
```

Tudo é feito no notebook [`atividade_ia_generativa.ipynb`](atividade_ia_generativa.ipynb), que já vem com o código funcionando. O trabalho do grupo é **adaptar o código ao seu tema**, executar, testar e registrar as observações pedidas em cada etapa.

---

## 3. Temas

Cada grupo escolhe **um** tema. O tema define o `SYSTEM_PROMPT` do assistente e as perguntas usadas nos testes.

| # | Tema | Sobre o que o assistente fala |
|---|------|-------------------------------|
| 1 | Assistente de casa inteligente | Automação residencial, sensores e dispositivos conectados |
| 2 | Suporte técnico de IoT | Diagnóstico de problemas comuns com ESP32, Arduino, Wi-Fi e sensores |
| 3 | Tutor de protocolos IoT | MQTT, HTTP, CoAP e LoRa explicados para iniciantes, com exemplos simples |
| 4 | Assistente de saúde e bem-estar (IoB) | Interpretação de dados de wearables (passos, sono, frequência cardíaca), **sem dar diagnóstico médico** |
| 5 | Consultor de agricultura inteligente | Irrigação, umidade do solo e estações meteorológicas conectadas |
| 6 | Assistente de mobilidade urbana | Cidades inteligentes, transporte, estacionamento e semáforos conectados |
| 7 | Consultor de eficiência energética | Dicas de consumo com base em medidores inteligentes e tomadas conectadas |
| 8 | Guia de privacidade e ética em IoB | Riscos da coleta de dados comportamentais, LGPD e consentimento do usuário |

> **Dica:** nos temas 4 e 8, usem o `system` também para colocar **limites** no assistente (por exemplo: "não dê diagnósticos médicos", "não dê aconselhamento jurídico"). Testem se o modelo respeita esses limites.

---

## 4. Conceitos importantes antes de começar

### 4.1 Mensagens e papéis (`role`)

A API de chat não recebe um texto solto. Ela recebe uma **lista de mensagens**, e cada mensagem tem duas informações:

- **`role`**: quem escreveu a mensagem
- **`content`**: o que foi escrito

```python
{"role": "user", "content": "O que é MQTT?"}
```

Existem três papéis:

| role | Quem é | Para que serve |
|------|--------|----------------|
| `system` | As instruções do assistente | Define comportamento, tom, idioma, tamanho das respostas e limites. Aparece uma vez, no início. O usuário final normalmente não vê. |
| `user` | A pessoa | O que o usuário digitou. |
| `assistant` | O modelo | O que o modelo já respondeu antes. |

Uma conversa completa fica assim:

```python
mensagens = [
    {"role": "system",    "content": "Você é um assistente de IoT. Responda em até 3 frases."},
    {"role": "user",      "content": "O que é MQTT?"},
    {"role": "assistant", "content": "MQTT é um protocolo leve de mensagens..."},
    {"role": "user",      "content": "E onde ele é usado?"},
]
```

**Por que o modelo precisa dessas etiquetas?**

No fim das contas, o modelo lê um único texto corrido. Antes de chegar ao modelo, a lista de mensagens é convertida num texto com marcações especiais, parecido com isto:

```
<|system|>    Você é um assistente de IoT. Responda em até 3 frases.
<|user|>      O que é MQTT?
<|assistant|> 
```

O modelo foi treinado com muitas conversas nesse formato e aprendeu o que cada marcação significa:

- texto depois de **system** é uma regra a seguir
- texto depois de **user** é algo a responder
- texto depois de **assistant** é o que ele mesmo disse, e é ali que ele continua escrevendo

Sem as etiquetas, o modelo não teria como separar instrução, pergunta e resposta anterior.

**Resumo:** o `system` configura o assistente, o `user` usa o assistente, e o `assistant` é o que o assistente já respondeu.

Uma forma de pensar: é como enviar para alguém o **print da conversa inteira do WhatsApp** a cada nova mensagem, com um bilhete no topo dizendo como essa pessoa deve se comportar (o `system`).

### 4.2 O modelo não tem memória

Cada chamada à API é **independente**. O modelo não lembra da chamada anterior.

Quem cria a "memória" é o **nosso código**: ele guarda as mensagens numa lista e reenvia a lista inteira a cada nova pergunta.

```python
historico.append({"role": "user", "content": entrada})        # guarda a pergunta
resposta = cliente.chat_completion(messages=historico)          # envia TUDO
historico.append({"role": "assistant", "content": resposta_txt}) # guarda a resposta
```

Se a lista for apagada, o modelo "esquece" a conversa. A Etapa 3 mostra isso na prática.

### 4.3 Diferenças no Gemini

A ideia é a mesma, mas os nomes mudam:

| Conceito | Hugging Face | Gemini |
|----------|--------------|--------|
| Instrução do assistente | mensagem com `"role": "system"` | parâmetro `system_instruction` |
| Mensagem do usuário | `"role": "user"` | `"role": "user"` |
| Resposta do modelo | `"role": "assistant"` | `"role": "model"` |
| Texto da mensagem | `"content": "..."` | `"parts": [{"text": "..."}]` |

Exemplo equivalente no Gemini:

```python
resposta = cliente_gemini.models.generate_content(
    model="gemini-2.5-flash-lite",
    contents=[
        {"role": "user",  "parts": [{"text": "O que é MQTT?"}]},
        {"role": "model", "parts": [{"text": "MQTT é um protocolo leve..."}]},
        {"role": "user",  "parts": [{"text": "E onde ele é usado?"}]},
    ],
    config=types.GenerateContentConfig(
        system_instruction="Você é um assistente de IoT. Responda em até 3 frases."
    ),
)
```

### 4.4 Parâmetros de geração

| Parâmetro | O que faz |
|-----------|-----------|
| `max_tokens` (HF) / `max_output_tokens` (Gemini) | Limite de tamanho da resposta, em tokens |
| `temperature` | Criatividade. Perto de 0: respostas mais previsíveis. Perto de 1: mais variadas |

### 4.5 Por que usar API e não baixar o modelo?

Na primeira parte da Aula 05, os modelos (GPT-2, BART) eram **baixados e executados no Colab** com `pipeline()`. Aqui usamos `InferenceClient` e `genai.Client`: o modelo roda **nos servidores** do Hugging Face ou do Google, e o nosso código só envia a pergunta e recebe a resposta. Isso permite usar modelos bem maiores sem precisar de GPU.

---

## 5. Pré-requisitos

- Conta Google (para usar o **Google Colab** e gerar a chave do **Gemini**)
- Conta no **Hugging Face** (gratuita): https://huggingface.co/join
- Conta no **GitHub** (para a entrega)

Não é preciso instalar nada no computador. Tudo roda no Colab.

---

## 6. Configuração passo a passo

### 6.1 Gerar o token do Hugging Face

1. Acesse https://huggingface.co/settings/tokens
2. Clique em **Create new token**
3. Escolha o tipo **Read** (ou, se usar **Fine-grained**, marque a permissão **Make calls to Inference Providers**)
4. Dê um nome (ex.: `aula-ia`) e clique em **Create token**
5. Copie o token (começa com `hf_`). Ele só aparece uma vez.

### 6.2 Gerar a chave do Gemini

1. Acesse https://aistudio.google.com/apikey
2. Clique em **Create API key**
3. Copie a chave

### 6.3 Abrir o notebook no Colab

**Opção A:** no Colab, vá em **Arquivo > Abrir notebook > GitHub**, cole o link deste repositório e selecione `atividade_ia_generativa.ipynb`.

**Opção B:** baixe o arquivo `.ipynb` e, no Colab, vá em **Arquivo > Fazer upload de notebook**.

Depois, salve uma cópia no seu Drive: **Arquivo > Salvar uma cópia no Drive**.

### 6.4 Cadastrar os tokens nos Secrets do Colab

1. No Colab, clique no **ícone de chave** na barra lateral esquerda
2. Clique em **Adicionar novo secret**
3. Crie dois secrets, com estes nomes **exatos**:

| Nome | Valor |
|------|-------|
| `HF_TOKEN` | token do Hugging Face |
| `GEMINI_API_KEY` | chave do Gemini |

4. Ative a opção **Acesso ao notebook** nos dois

### 6.5 Testar a configuração

Execute as células da seção **0. Configuração**. A saída esperada é:

```
HF_TOKEN ok
GEMINI_API_KEY ok
```

Se aparecer `ERRO`, revise o passo 6.4.

### 6.6 Personalizar o `SYSTEM_PROMPT`

Ainda na seção 0, troque o `SYSTEM_PROMPT` de exemplo (casa inteligente) por um que descreva o assistente do tema do grupo. Um bom `system` costuma dizer:

- **quem** o assistente é
- **sobre o que** ele fala
- **como** responde (idioma, tamanho, tom)
- **o que ele não deve fazer**

```python
SYSTEM_PROMPT = """Você é um tutor de protocolos IoT para iniciantes.
Explique MQTT, HTTP, CoAP e LoRa com exemplos simples do dia a dia.
Responda sempre em português, em no máximo 5 frases.
Se a pergunta não for sobre protocolos IoT, diga educadamente que não pode ajudar."""
```

---

## 7. As etapas da atividade

Em todas as etapas, altere os trechos marcados com `# >>> PERSONALIZE` e responda as **Observações do grupo** nas células de texto.

### Etapa 1: Assistente com personalidade

**O que fazer:**
1. Crie 3 personalidades (3 `system` diferentes) no dicionário `personalidades`
2. Escreva uma pergunta sobre o tema do grupo
3. Execute e compare as 3 respostas

**Código principal:**

```python
def perguntar_hf(pergunta, system, temperatura=0.7):
    mensagens = [
        {"role": "system", "content": system},
        {"role": "user",   "content": pergunta},
    ]
    resposta = cliente_hf.chat_completion(messages=mensagens, max_tokens=300, temperature=temperatura)
    return resposta.choices[0].message.content
```

**O que observar:** a mesma pergunta gera respostas bem diferentes só mudando o `system`.

### Etapa 2: Hugging Face x Gemini

**O que fazer:**
1. Escreva 3 perguntas sobre o tema do grupo na lista `perguntas`
2. Execute: cada pergunta é enviada aos dois modelos com o mesmo `SYSTEM_PROMPT`
3. Compare as respostas

**Código principal (Gemini):**

```python
def perguntar_gemini(pergunta, system):
    resposta = cliente_gemini.models.generate_content(
        model=MODELO_GEMINI,
        contents=pergunta,
        config=types.GenerateContentConfig(system_instruction=system, max_output_tokens=300),
    )
    return resposta.text
```

**O que observar:** os dois modelos seguiram as regras do `system` (idioma, tamanho, tema)? Em qual pergunta a diferença foi maior?

### Etapa 3: Chat com memória

**O que fazer:**
1. Execute o chat e converse com o assistente
2. Faça o teste de memória:
   - diga seu nome
   - pergunte "qual é o meu nome?"
   - digite `limpar`
   - pergunte de novo "qual é o meu nome?"
3. Use o comando `historico` para ver quantas mensagens estão guardadas
4. Digite `sair` para encerrar

**O que observar:** depois do `limpar`, o assistente não sabe mais o nome. A memória estava na lista `historico`, não no modelo.

> **Atenção:** enquanto o chat estiver rodando, a célula fica ocupada esperando o `input()`. Digite `sair` para liberar.

### Etapa 4: Interface web com Gradio

**O que fazer:**
1. Ajuste `title`, `description` e `examples` para o tema do grupo
2. Execute a célula: o Colab mostra a interface e um **link público** (`share=True`)
3. Converse usando os dois modelos (seletor **Modelo**)
4. Tire um print da interface funcionando

**Como funciona:** a função `responder` recebe a mensagem, o histórico (que o Gradio guarda) e o modelo escolhido. Para o Hugging Face, monta a lista com `system`/`user`/`assistant`. Para o Gemini, converte `assistant` em `model` e passa o `system` em `system_instruction`.

**O que observar:** troquem de modelo no meio da conversa. O assistente continua lembrando do que foi dito, porque o histórico é do Gradio (do nosso código), e não do modelo.

> Para parar a interface, interrompa a célula no Colab.

### Etapa 5 (Bônus): API com FastAPI

**O que fazer:**
1. Execute a célula `%%writefile app.py` (cria o arquivo do servidor)
2. Execute a célula que inicia o servidor em segundo plano
3. Execute a célula de teste, que faz um `POST` em `/chat`
4. Encerre o servidor na última célula

**Fluxo:**

```
[Frontend / App / Dispositivo]  --POST /chat {"mensagem": "..."}-->  [FastAPI]  -->  Hugging Face API
                                <--------- {"resposta": "..."} ------------
```

**Por que um backend?** O token nunca pode ficar no frontend. O backend guarda o token, aplica o `system` e permite trocar o modelo sem mexer no frontend.

**Para rodar no seu computador (opcional):**

```bash
pip install -r requirements.txt
export HF_TOKEN="seu_token"            # Windows (PowerShell): $env:HF_TOKEN="seu_token"
uvicorn app:app --reload
```

Depois acesse http://localhost:8000/docs para testar o endpoint pelo navegador.

---

## 8. Entrega

**Formato:** link de um repositório público no GitHub, criado pelo grupo.

**Prazo:** _(definido pelo professor)_

**O repositório do grupo deve conter:**

```
nome-do-repositorio/
├── README.md                          # README do grupo (modelo abaixo)
├── atividade_ia_generativa.ipynb      # notebook EXECUTADO, com as saídas visíveis
└── prints/
    └── gradio.png                     # print da interface da Etapa 4
```

**Modelo de README do grupo:**

```markdown
# Assistente de IA: <nome do tema>

## Integrantes
- Nome (RM)
- Nome (RM)

## Tema
<tema escolhido e o que o assistente faz>

## System prompt usado
<cole aqui o SYSTEM_PROMPT do grupo>

## Etapas realizadas
- [ ] Etapa 1: Assistente com personalidade
- [ ] Etapa 2: Hugging Face x Gemini
- [ ] Etapa 3: Chat com memória
- [ ] Etapa 4: Interface Gradio
- [ ] Etapa 5 (bônus): API FastAPI

## Interface
![Interface Gradio](prints/gradio.png)
```

> **Antes de enviar:** confira se **nenhum token** aparece no notebook ou no README (veja a seção 10).

---

## 9. Erros comuns

| Erro / sintoma | Causa provável | Solução |
|----------------|----------------|---------|
| `ERRO: adicione HF_TOKEN nos Secrets` | Secret não criado, nome diferente ou sem acesso ao notebook | Revise o passo 6.4. O nome precisa ser exatamente `HF_TOKEN` / `GEMINI_API_KEY` |
| `SecretNotFoundError` | Mesmo caso acima | Idem |
| `401 Unauthorized` (HF) | Token inválido ou sem permissão de inferência | Gere um novo token (passo 6.1) |
| `402 Payment Required` (HF) | Créditos gratuitos de inferência do mês esgotados | Use o token de outro integrante ou teste com o Gemini |
| `429` / `RESOURCE_EXHAUSTED` (Gemini) | Limite de requisições do plano gratuito | Aguarde alguns minutos e tente de novo |
| `404` / modelo não encontrado | Nome do modelo digitado errado ou modelo descontinuado | Confira `MODELO_HF` e `MODELO_GEMINI` |
| Célula do chat não termina | O `input()` está esperando digitação | Digite `sair` ou interrompa a célula |
| Erro ao desempacotar o histórico no Gradio (`too many values to unpack`) | Código antigo que trata o histórico como tuplas | Use a função `responder` deste notebook, que trata o formato de mensagens |
| `Address already in use` (FastAPI) | Servidor anterior ainda rodando | Execute a célula que encerra o servidor ou reinicie o ambiente |
| `NameError` | Células executadas fora de ordem | **Ambiente de execução > Executar tudo**, ou execute a partir da seção 0 |

---

## 10. Segurança dos tokens

- **Nunca** escreva o token direto no código (`HF_TOKEN = "hf_..."`).
- Use sempre os **Secrets do Colab** ou **variáveis de ambiente**.
- **Nunca** faça commit de arquivos `.env` ou de tokens. Este repositório inclui um `.gitignore` que ignora `.env`.
- Se um token vazar, **revogue** na página onde ele foi criado e gere outro.

---

## 11. Estrutura deste repositório

```
.
├── README.md                        # este guia
├── atividade_ia_generativa.ipynb    # notebook base da atividade
├── requirements.txt                 # dependências (para rodar fora do Colab)
└── .gitignore
```

---

## 12. Referências

- Hugging Face, Inference Providers: https://huggingface.co/docs/inference-providers
- Hugging Face, `InferenceClient`: https://huggingface.co/docs/huggingface_hub/guides/inference
- Google Gemini API (Python): https://ai.google.dev/gemini-api/docs
- Gemini, instruções de sistema: https://ai.google.dev/gemini-api/docs/text-generation
- Gradio, `ChatInterface`: https://www.gradio.app/docs/gradio/chatinterface
- FastAPI: https://fastapi.tiangolo.com/

<img width="994" height="455" alt="{7DC3AA65-D85D-489A-A99F-62B23041874D}" src="https://github.com/user-attachments/assets/ea204494-ecbf-423e-8616-e26639dab726" />
<img width="970" height="445" alt="{81608666-397E-4574-8E7F-388C037C4F1F}" src="https://github.com/user-attachments/assets/466dc42a-d7e4-4bbc-bf89-6c69bc1c3f31" />

