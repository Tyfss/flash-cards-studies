# Flash Cards Studies

Aplicação web single-file que converte anotações de aula (fotos, PDF, DOCX, PPTX ou vídeo/áudio) em flashcards de estudo usando IA, com exportação em CSV compatível com o Anki e opção de baixar um site offline para estudar sem internet.

**Site em produção:** https://flashcardsstudies.netlify.app
**Arquivos:** `index.html` (site) e `_headers` (cabeçalhos de segurança da Netlify)

---

## 1. Visão geral

| | |
|---|---|
| **Tipo** | Single Page Application (SPA), arquivo HTML único |
| **Backend** | Nenhum, 100% client-side |
| **Custo de operação** | Zero (hospedagem e IA em camada gratuita) |
| **Autenticação** | Nenhuma, usuário cola sua própria chave de API |
| **Persistência** | Nenhuma, estado existe só em memória durante a sessão |

O app roda inteiramente no navegador do usuário. Não existe servidor próprio: a extração de texto acontece localmente (OCR, parsing de PDF/DOCX/PPTX, transcrição de vídeo/áudio) e a geração dos flashcards é feita por uma chamada direta do navegador à API do Google Gemini, usando uma chave que o próprio usuário fornece.

---

## 2. Funcionalidades

- Upload múltiplo de arquivos: imagens (JPG/PNG/WEBP), PDF, DOCX, PPTX, vídeo e áudio (MP4, MOV, WEBM, MP3, WAV, M4A, OGG)
- Extração de texto local, sem enviar arquivo a nenhum servidor:
  - Imagens → OCR (português + inglês)
  - PDF → extração de texto nativo (não funciona em PDF escaneado sem camada de texto)
  - DOCX → extração de texto bruto
  - PPTX → leitura do XML interno de cada slide
  - Vídeo/áudio (arquivo local) → transcrição de fala local via modelo Whisper rodando no navegador (até ~25 min por arquivo)
- **Link do YouTube**: cola a URL e o próprio Gemini assiste e ouve o vídeo direto da fonte, sem o site baixar nada. Não tem o limite de 25 minutos do upload local (o vídeo é processado pelo Google, não pelo navegador da pessoa)
- Geração de flashcards via IA (Google Gemini), com:
  - Quantidade configurável (3 a 60 por geração)
  - Rodadas automáticas de complementação caso a IA devolva menos cartões que o pedido
  - Fallback entre modelos Gemini caso o modelo principal esteja indisponível
  - Pode combinar texto extraído de arquivos com vídeos do YouTube na mesma geração
- Criação e edição manual de flashcards, a qualquer momento, antes ou depois da geração por IA. Cartões manuais nunca são substituídos pelos gerados, apenas somados
- Interface de cartões "viráveis" (frente/verso), com edição inline
- Exportação em CSV (UTF-8, formato `"pergunta","resposta"`) compatível com importação no Anki
- Exportação de site offline para estudo: gera um segundo arquivo HTML autocontido, sem dependências externas, com os flashcards atuais embutidos
- Botão "Esquecer" para apagar a chave de API da memória a qualquer momento, sem precisar recarregar a página

### O que não é suportado, de propósito

**Spotify e outros podcasts por link não são aceitos.** A API do Gemini não tem uma opção de "processar direto por link" para áudio como tem para vídeo do YouTube: ela só aceita áudio como arquivo enviado. Some a isso o mesmo bloqueio técnico de CORS para baixar de uma plataforma de streaming sem servidor no meio, e a questão de Termos de Uso dessas plataformas quanto a download automatizado.

O caminho suportado para podcast: baixar o áudio por um meio que o usuário já tem permissão de usar e enviar esse arquivo local, que aí sim é transcrito no navegador.

**YouTube é diferente e É suportado**, por link direto: é um recurso oficial da própria API do Gemini (não uma automação por fora), então não esbarra em CORS nem em Termos de Uso de terceiros. Limitações da própria Google que valem aqui: o vídeo precisa ser público, e a camada gratuita da API tem um teto de 8 horas de vídeo do YouTube processadas por dia.

---

## 3. Stack técnica

| Camada | Tecnologia | Função |
|---|---|---|
| UI | HTML + CSS + JavaScript (vanilla, sem framework) | Interface e lógica |
| OCR | [Tesseract.js](https://github.com/naptha/tesseract.js) 5.0.4 | Leitura de texto em imagens |
| PDF | [PDF.js](https://mozilla.github.io/pdf.js/) 3.11.174 | Extração de texto de PDF |
| DOCX | [Mammoth.js](https://github.com/mwilliamson/mammoth.js) 1.6.0 | Extração de texto de Word |
| PPTX | [JSZip](https://stuk.github.io/jszip/) 3.10.1 | Leitura do XML dentro do .pptx |
| Transcrição de mídia | [Transformers.js](https://huggingface.co/docs/transformers.js) 3.0.0 + modelo `Xenova/whisper-tiny` | Fala → texto, rodando localmente via WebAssembly |
| IA de flashcards | [Google Gemini API](https://ai.google.dev/) (`generateContent`) | Geração das perguntas e respostas |
| Tipografia | Google Fonts: Sora (títulos), Inter (texto) | Design |
| Hospedagem | Netlify (plano gratuito) | Deploy estático |

Todas as bibliotecas são carregadas via CDN (cdnjs.cloudflare.com e cdn.jsdelivr.net). Não há `npm install` nem processo de build: o arquivo é aberto e executado diretamente pelo navegador.

---

## 4. Transcrição de vídeo/áudio (como funciona)

1. O arquivo é lido localmente e decodificado com a Web Audio API (`decodeAudioData`), sem precisar de nenhuma biblioteca de conversão externa.
2. O áudio é reamostrado para 16kHz mono (formato que o modelo espera) usando `OfflineAudioContext`.
3. O modelo `Xenova/whisper-tiny` (carregado via Transformers.js) transcreve o áudio em blocos de 30 segundos com sobreposição de 5 segundos, para lidar com arquivos mais longos sem perder contexto entre os cortes.
4. O texto transcrito entra no mesmo fluxo dos outros formatos: vira insumo para a geração de flashcards.

**Limites conscientes:**
- Arquivos acima de 25 minutos são recusados com uma mensagem clara, em vez de travar o navegador por horas rodando um modelo de IA no processador do usuário.
- Na primeira vez que a pessoa usa essa função, o navegador baixa o modelo (dezenas de MB). Fica em cache do navegador depois disso.
- É uma funcionalidade mais pesada computacionalmente que as outras: pode ser lenta em computadores mais fracos.

---

## 5. Segurança

### O que é tecnicamente possível, e o que não é

Um site 100% client-side **não consegue esconder de quem usa o navegador um valor que a própria pessoa digitou**, incluindo a chave de API. Isso não é uma falha deste projeto: é uma característica de como navegadores funcionam. Qualquer JavaScript rodando no navegador de alguém é, por definição, inspecionável por essa mesma pessoa via DevTools. A única forma de esconder de verdade uma chave de quem usa o site é ter um servidor no meio guardando essa chave (ver seção 5.4).

Dado isso, o que este projeto protege de fato:

### 5.1 Proteção contra roubo de chave por terceiros (Content-Security-Policy)

O `index.html` inclui uma política de `Content-Security-Policy` restringindo:
- De quais domínios o site pode carregar script (`script-src`): só os CDNs usados (cdnjs, jsdelivr) e o próprio site, usando hash SHA-256 do conteúdo exato dos blocos de script inline (não é `'unsafe-inline'`)
- Para onde o site pode enviar requisições de rede (`connect-src`): só a API do Gemini e os hosts de onde os modelos/bibliotecas são baixados
- Bloqueio de `<iframe>` embutindo o site, de plugins (`object-src`), e de formulários apontando para fora (`form-action`)

Isso reduz o risco real: se algum dia um CDN de terceiro for comprometido e tentar injetar um script malicioso, o navegador recusa rodar ou conectar a qualquer domínio fora da lista, mesmo que o script consiga entrar na página.

> Se o JavaScript do `index.html` for editado no futuro, os hashes `sha256-...` no CSP precisam ser recalculados (eles descrevem o conteúdo exato dos blocos de script, byte a byte). Comando para recalcular:
> ```python
> import re, hashlib, base64
> html = open('index.html', encoding='utf-8').read()
> scripts = re.findall(r'<script>((?:(?!</script>).)*)</script>', html, re.S)
> print('sha256-' + base64.b64encode(hashlib.sha256(scripts[0].encode()).digest()).decode())
> ```

### 5.2 Cabeçalhos HTTP (`_headers`)

O arquivo `_headers`, na raiz do deploy, instrui a Netlify a enviar cabeçalhos que só fazem efeito vindos do servidor (não podem ser feitos por `<meta>`):
- `X-Content-Type-Options: nosniff`, evita que o navegador tente "adivinhar" um tipo de arquivo diferente do declarado
- `X-Frame-Options: DENY`, reforça o bloqueio de embutir o site em iframe de outro domínio
- `Strict-Transport-Security`, força HTTPS em todas as visitas futuras
- `Permissions-Policy`, desliga o acesso a câmera, microfone, geolocalização e pagamentos, que o site não usa

### 5.3 Minimização de exposição da chave

- Nunca salva em `localStorage`, `sessionStorage` ou cookie: existe só em uma variável JavaScript em memória
- Campo `type="password"`, não aparece em texto puro na tela
- Botão **"Esquecer"** para apagar a chave da memória a qualquer momento, sem esperar recarregar a página
- Nenhum `console.log` ou envio da chave para qualquer destino além de `generativelanguage.googleapis.com`, direto do navegador

### 5.4 Se quiser esconder a chave de verdade

A única forma real é um proxy no servidor: uma função (ex: Netlify Functions, tem plano gratuito) guarda a chave do lado do servidor, e o navegador fala só com essa função, nunca direto com a Google. Ainda gratuito, mas muda o modelo: a chave passa a ser do dono do site, compartilhada entre todos os visitantes, com risco de esgotar a cota gratuita se alguém abusar (precisaria de um limite de uso por pessoa/dia). **Decisão tomada no projeto: manter o modelo atual, cada pessoa com sua própria chave**, por ser mais simples e não ter esse risco de custo compartilhado.

---

## 6. Arquitetura e fluxo de dados

```
Usuário                Navegador (client-side)              Google Gemini API
   │                          │                                    │
   ├─ envia arquivo(s) ──────>│                                    │
   │                          ├─ detecta tipo (fileKind)           │
   │                          ├─ extrai texto (OCR / PDF / DOCX /  │
   │                          │   PPTX / transcrição de mídia,     │
   │                          │   tudo local)                      │
   │                          │                                    │
   ├─ cola chave de API ─────>│                                    │
   ├─ clica "Gerar" ─────────>│                                    │
   │                          ├─ monta prompt + texto extraído ───>│
   │                          │<── retorna JSON com flashcards ────┤
   │                          ├─ valida quantidade, complementa    │
   │                          │   se necessário (até 4 rodadas)    │
   │                          ├─ soma aos cartões manuais já       │
   │                          │   existentes (nunca substitui)     │
   │                          ├─ renderiza cartões editáveis       │
   │<── exibe flashcards ─────┤                                    │
   ├─ edita / exporta CSV ou │                                    │
   │  site offline ──────────>│                                    │
   │<── download do arquivo ──┤                                    │
```

---

## 7. Estrutura do código (`index.html`)

| Bloco | Conteúdo |
|---|---|
| `<meta http-equiv="Content-Security-Policy">` | Política de segurança do navegador (ver seção 5.1) |
| `<style>` | Design system (variáveis CSS), layout responsivo, animação de flip dos cartões |
| Seção 1, Enviar material | Dropzone de arquivos, lista de arquivos com status de processamento |
| Seção 2, Configurar geração | Campo de chave de API (com botão Esquecer), quantidade de cartões, botão gerar |
| Seção 3, Seus flashcards | Grid de flashcards, estado vazio, botões de adicionar, exportar CSV e exportar site offline |
| `<script type="module">` | Carrega o Transformers.js e expõe `window.__getTranscriber` para o script principal |
| `state` (JS) | `files[]` (arquivos e texto extraído) e `flashcards[]` (cartões atuais) |
| `fileKind()` / `extractImage()` / `extractPdf()` / `extractDocx()` / `extractPptx()` / `extractMedia()` | Detecção de tipo e extração de texto por formato |
| `decodeToMono16k()` | Decodifica e reamostra áudio/vídeo para o formato que o Whisper espera |
| `addYoutubeLink()` | Valida e adiciona um link do YouTube à lista, sem download nenhum |
| `buildPrompt()` / `callGeminiModel()` / `callWithFallback()` / `generateFlashcardsFromText()` | Integração com a API do Gemini, incluindo fallback de modelo e rodadas de complementação |
| `renderCards()` / `renderFileList()` | Renderização da interface a partir do `state` |
| `csvEscape()` / listener do `exportBtn` | Geração e download do CSV |
| `buildOfflineSiteHtml()` / listener do `offlineBtn` | Geração e download do site de estudo offline |

---

## 8. Integração com a API do Gemini

- **Endpoint:** `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent`
- **Autenticação:** header `x-goog-api-key`, com a chave fornecida pelo usuário
- **Modelos, em ordem de tentativa** (`MODEL_CANDIDATES`):
  1. `gemini-flash-latest`, alias mantido pelo Google que sempre aponta ao Flash mais recente
  2. `gemini-2.5-flash`
  3. `gemini-2.0-flash`
  4. `gemini-flash-lite-latest`
- O fallback só avança para o próximo modelo em erro `404` (modelo indisponível). Erros de chave inválida ou cota excedida interrompem imediatamente, sem tentar outro modelo.
- **Formato de resposta:** forçado via `generationConfig.responseMimeType = "application/json"`.
- **Limite de tokens de saída:** calculado dinamicamente (~120 tokens por cartão + margem), até um teto de 8192.
- **Complementação automática:** se a quantidade retornada for menor que a pedida, o app repete a chamada só para os cartões restantes (até 4 rodadas), evitando repetição via lista de perguntas já geradas.
- **Vídeo do YouTube:** cada link vira uma parte `{"fileData": {"fileUri": "<url>"}}` somada às `parts` da requisição, junto do texto do prompt. A API busca e processa o vídeo diretamente, sem o navegador da pessoa baixar ou tocar no conteúdo.

---

## 9. Exportação do site offline

O botão **"Baixar site offline para estudar"** gera, no próprio navegador, um segundo arquivo HTML (`flashcards-offline.html`) com:

- Os flashcards atuais embutidos diretamente no código, como dados estáticos
- CSS e JavaScript próprios, totalmente inline, sem nenhuma dependência de CDN ou fonte externa
- Modo de estudo com um cartão por vez: virar, avançar, voltar e embaralhar
- Nenhuma chamada de rede: funciona abrindo o arquivo direto do computador, sem internet

Esse arquivo é independente do site principal. É um "retrato" dos flashcards no momento do download, sem sincronizar com edições futuras feitas no site.

---

## 10. Limites conhecidos

| Limite | Valor | Motivo |
|---|---|---|
| Cartões por geração | 60 | Evitar estourar a cota gratuita da API em uma única chamada |
| Texto enviado à IA por geração | ~30.000 caracteres | Limite de contexto/custo do prompt |
| PDF escaneado (sem texto nativo) | Não suportado | PDF.js só lê texto real, não faz OCR de imagem dentro do PDF. Nesse caso, enviar como foto |
| PPTX só com imagens (sem texto) | Não suportado | Não há OCR aplicado ao conteúdo do slide, só extração de texto |
| Vídeo/áudio local acima de 25 minutos | Recusado | Transcrição local ficaria impraticavelmente lenta no navegador |
| Link do YouTube | Suportado, até 8h de vídeo processadas por dia (camada gratuita) | Limite da própria API do Gemini, vídeo precisa ser público |
| Link do Spotify ou outro podcast por URL | Não suportado | A API do Gemini não tem essa opção para áudio, só para YouTube; enviar como arquivo local |
| Armazenamento entre sessões | Inexistente | Nenhum dado é salvo, recarregar a página apaga cartões e chave |
| Site offline exportado | Estático | Não recebe atualizações automáticas dos cartões editados depois do download |
| Ocultar a chave de API de quem a digitou | Não é possível (client-side puro) | Ver seção 5 |

---

## 11. Deploy

Hospedado na **Netlify** (plano gratuito), via drag-and-drop, sem build step e sem repositório Git.

**Publicar do zero:**
1. Criar conta gratuita na Netlify
2. Acessar [app.netlify.com/drop](https://app.netlify.com/drop)
3. Arrastar os dois arquivos juntos: `index.html` e `_headers`

**Atualizar versão existente:**
1. Abrir o site no painel da Netlify, aba **Deploys**
2. Arrastar os dois arquivos (`index.html` e `_headers`) na área de drag-and-drop
3. Publicação automática, em segundos

> O arquivo principal precisa se chamar `index.html` para ser servido como página inicial do domínio. O `_headers` precisa estar na raiz, junto dele, para a Netlify aplicar os cabeçalhos de segurança.

---

## 12. Como rodar localmente

Não há instalação nem dependências. Basta abrir o arquivo diretamente:

```bash
open index.html          # macOS
start index.html         # Windows
```

Ou servir por um servidor estático simples, útil se o navegador bloquear algum recurso ao abrir via `file://`:

```bash
python3 -m http.server 8000
# acessar http://localhost:8000
```

> Rodando via `file://` ou servidor local simples, o `_headers` não tem efeito (só funciona em produção, na Netlify). O CSP via `<meta>` continua valendo mesmo local.

---

## 13. Roadmap / possíveis evoluções

- OCR de PDF escaneado (renderizar páginas como imagem + Tesseract.js)
- Exportação direta em `.apkg` (pacote nativo do Anki), além do CSV
- Seleção de idioma de saída dos flashcards
- Modo de estudo com repetição espaçada dentro do próprio site
- Proxy server-side opcional, para quem quiser esconder a chave de verdade em troca de compartilhar cota (ver seção 5.4)
