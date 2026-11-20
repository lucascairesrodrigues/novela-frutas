# 🍓 Novela das Frutas – Pipeline de IA (n8n + Python)

Projeto que gera episódios curtos de uma novela de frutas de ponta a ponta, encadeando várias IAs.

## Arquitetura (alvo)

```
Tema → [n8n] Claude (roteiro JSON)
          → API de imagem (cenas)
          → API de vídeo (animação)
          → ElevenLabs (vozes)
          → [Python/FFmpeg] montagem → episodio.mp4
```

## Fases

- [ ] **Fase 1 – Roteirista:** n8n chama a API do Claude e salva o roteiro em JSON
- [ ] Fase 2 – Ilustrador: gera imagens das cenas com personagens consistentes
- [ ] Fase 3 – Dublador + Editor: vozes e montagem do vídeo (serviço Python)
- [ ] Fase 4 – Animador: troca imagens estáticas por vídeos
- [ ] Fase 5 – Extras: interface, fila, cache, controle de custos

---

## Fase 1 – passo a passo

### 1. Pré-requisitos
- Docker Desktop instalado
- Uma chave da API do Claude: crie em https://console.anthropic.com (é cobrado por uso; um roteiro custa centavos)

### 2. Subir o n8n
```bash
mkdir output
docker compose up -d
```
Abra http://localhost:5678 e crie sua conta local.

### 3. Guardar a chave com segurança
No n8n: **Credentials → Add credential → Header Auth**
- Name: `x-api-key`
- Value: sua chave da API do Claude

Nunca coloque a chave direto no fluxo, porque ela iria parar no GitHub quando você exportar o workflow.

### 4. Montar o fluxo (4 nós)

**Nó 1 – Manual Trigger**
Para testar clicando em "Execute workflow".

**Nó 2 – Set** (nome: `Tema`)
Campo `tema` (string), por exemplo: `Episódio 1: Morgana descobre que está grávida e o ex volta para a cidade`.

**Nó 3 – HTTP Request** (nome: `Roteirista Claude`)
- Method: `POST`
- URL: `https://api.anthropic.com/v1/messages`
- Authentication: *Generic Credential Type → Header Auth* → a credencial do passo 3
- Send Headers: ligado
  - `anthropic-version`: `2023-06-01`
  - `content-type`: `application/json`
- Send Body: JSON

```json
{
  "model": "claude-sonnet-5-5",
  "max_tokens": 4000,
  "system": "COLE AQUI O CONTEÚDO DE roteirista.md",
  "messages": [
    { "role": "user", "content": "{{ $json.tema }}" }
  ]
}
```

Dica: em vez de colar o prompt no corpo, você pode guardá-lo em outro nó **Set** e referenciar com uma expressão. Fica mais limpo de manter.

**Nó 4 – Code** (nome: `Parse JSON`, linguagem JavaScript)
```javascript
const texto = $input.first().json.content[0].text;
const roteiro = JSON.parse(texto);
return [{ json: roteiro }];
```

### 5. Testar
Execute o fluxo. No nó `Parse JSON` você deve ver o episódio com as cenas separadas.
Se der erro de JSON, o Claude provavelmente escreveu algo fora do JSON: reforce a instrução no prompt ou limpe o texto antes do `JSON.parse`.

### 6. Salvar no GitHub
No n8n: menu do workflow → **Download** → salve como `workflows/fase1-roteirista.json` no repositório.

---

## Estrutura do repositório

```
novela-frutas/
├── docker-compose.yml
├── README.md
├── prompts/
│   └── roteirista.md
├── workflows/          # exports do n8n
├── editor/             # serviço Python (Fase 3)
├── assets/personagens/ # imagens de referência
└── output/             # episódios gerados (no .gitignore)
```
