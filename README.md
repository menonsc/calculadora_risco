## Guia de Integração: Envio de Leads e Resultados ao N8N (com fallbacks)

Este guia documenta as alterações aplicadas no `index.html` desta calculadora e como replicá-las em qualquer outra ferramenta interativa estática (HTML/JS puro).

### Objetivo
- Capturar respostas do usuário (incluindo score e metadados) e enviar ao N8N via Webhook.
- Tornar o envio resiliente a CORS e a cenários com bloqueios, usando múltiplas estratégias de fallback.
- Enviar os dados de forma organizada (campos separados) usando `application/x-www-form-urlencoded`.

### Principais componentes adicionados
1) Estrutura de perguntas e cálculo de score
- Array `questions`, controle `currentQuestionIndex`, `score`, `userAnswers` e `answersDetailed`.

2) Renderização e captura das respostas
- Funções `renderQuestion()` e `selectAnswer(value, question, answerText)` – acumulam `score`, preenchem `userAnswers` e `answersDetailed`.

3) Tela de resultados e interpretação
- Funções `showResults()` e `getWorstFactors()` para exibir score, categoria e interpretação.

4) Coleta de metadados úteis
- `getUTMParams()` (UTMs), `page` (URL/referrer), `device` (user agent, idioma, resolução).

5) Envio ao N8N com múltiplos fallbacks
- Constante `N8N_WEBHOOK_URL` com a URL do webhook.
- `buildFormBody(payload)` para montar corpo `application/x-www-form-urlencoded`, “achatando” objetos e arrays para chaves separadas (ex.: `answers_text[Você é fumante?]`, `answers_detailed[0][question]`).
- `sendDataToWebhook(payload)` com estratégia em cascata:
  - fetch CORS (POST + `application/x-www-form-urlencoded`) – simple request, evita preflight.
  - fetch `no-cors` (fire-and-forget) – mesma codificação.
  - `navigator.sendBeacon` – mesma codificação.
  - formulário oculto + iframe – última linha de defesa em cenários restritos de navegador.

6) UX de envio
- Feedback visual com `showEnvioStatus(message, type)` (sucesso/erro) e estados no botão de submit.

### Estrutura do payload
Campos enviados (exemplos):
- `name`, `email`, `consent`
- `score`, `score_category`, `total_questions`
- `answers_text[pergunta] = resposta`
- `answers_detailed[index][question|answer|value]`
- `utm[utm_source|utm_medium|utm_campaign|utm_term|utm_content]`
- `page[url|referrer]`
- `device[user_agent|language|screen[width|height]]`
- `timestamp`

Observação: objetos e arrays são achatados em chaves legíveis no `x-www-form-urlencoded`, facilitando o mapeamento no N8N sem precisar de parsing adicional.

### Passo a passo para replicar em outro projeto
1. Copie os blocos de código
- Copie do `index.html` as seguintes partes: perguntas/score, `answersDetailed`, `getUTMParams`, `showScreen`/renderização, `showResults`/`getWorstFactors`, `buildFormBody`, `sendDataToWebhook`, `postViaHiddenForm`, `showEnvioStatus`/`createStatusDiv`, e o listener de `submit` do formulário que monta o `payload`.

2. Ajuste o webhook
- Altere `N8N_WEBHOOK_URL` para o endpoint do seu fluxo no N8N.

3. Valide o formulário
- Garanta que os inputs do formulário (`name`, `email`, `consent`) existam e que a lógica de pontuação rode antes do submit.

4. Teste localmente
- Abra o HTML no navegador, preencha e envie. Se o servidor N8N não estiver com CORS, o envio ainda ocorrerá pelos fallbacks (`no-cors`, `sendBeacon`, formulário oculto).

5. Mapeie no N8N
- No nó Webhook, verifique o `Body` entrando como `Form-URL Encoded`.
- Os campos achatados facilitarão o mapeamento diretamente para nós seguintes (ex.: Google Sheets, CRM etc.).

### Boas práticas de CORS (opcional, mas recomendado)
No proxy/Nginx/N8N, habilite:
- `Access-Control-Allow-Origin: https://SEU-DOMINIO`
- `Access-Control-Allow-Methods: POST, OPTIONS`
- `Access-Control-Allow-Headers: Content-Type`

### GitHub Pages (deploy automático)
- Workflow em `.github/workflows/deploy.yml` publica o conteúdo do repositório no GitHub Pages a cada push na `main`.
- Em Settings → Pages, selecione a fonte “GitHub Actions”.

### Customizações comuns
- Mudar chaves das respostas: adapte o “achatamento” em `buildFormBody` para nomes do seu padrão (ex.: `respostas[1].pergunta`).
- Adicionar mais metadados: inclua chaves no `payload` (o builder trata objetos/arrays automaticamente).

### Segurança e privacidade
- Envie apenas dados necessários. Evite dados sensíveis que não precisa armazenar.
- Se usar domínio público, limite CORS ao seu domínio.

### Suporte
Se precisar, copie trechos do `index.html` deste repo e cole no seu projeto, ajustando os seletores de DOM e o `N8N_WEBHOOK_URL`.


