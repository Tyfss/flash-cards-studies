# Flash Cards Studies

[![Netlify Status](https://api.netlify.com/api/v1/badges/01ff4d3b-94f9-452b-ae09-e6083a2b4b19/deploy-status)](https://app.netlify.com/projects/flashcardsstudies/deploys)

Site gratuito, 100% no navegador, que transforma anotações de aula em flashcards de estudo usando IA. Sem servidor, sem custo, sem cadastro.

**Site em produção:** https://flashcardsstudies.netlify.app

## O que faz

Envie fotos, PDF, DOCX, PPTX, vídeo/áudio local, ou cole um link do YouTube. O texto é extraído no seu navegador (OCR, leitura de PDF/DOCX/PPTX, transcrição de fala) e enviado para a API gratuita do Google Gemini, que gera pares de pergunta e resposta. Os cartões podem ser editados, criados manualmente, exportados em CSV pronto pro Anki, ou baixados como um site offline para estudar sem internet.

## Funcionalidades

- Upload de imagens, PDF, DOCX, PPTX, vídeo e áudio
- Transcrição de vídeo/áudio local direto no navegador (Whisper via WebAssembly, sem enviar a nenhum servidor)
- Link do YouTube processado nativamente pela API do Gemini (sem download)
- Geração de flashcards configurável (3 a 60 por vez), com complementação automática
- Criação e edição manual de cartões, somados aos gerados por IA
- Exportação em CSV compatível com Anki
- Exportação de site offline autocontido para estudar sem internet
- Content-Security-Policy com hash real do código, cabeçalhos de segurança HTTP

Detalhes técnicos completos, arquitetura, limites conhecidos e a seção de segurança: [`docs/DOCUMENTACAO.md`](docs/DOCUMENTACAO.md).

## Rodar localmente

Não tem instalação nem dependências. Só abrir o arquivo:

```bash
open index.html          # macOS
start index.html         # Windows
```

Ou por um servidor estático simples:

```bash
python3 -m http.server 8000
```

## Deploy

Hospedado na [Netlify](https://netlify.com), plano gratuito. Dois jeitos:

**Manual (arrastar e soltar):** em [app.netlify.com/drop](https://app.netlify.com/drop), arraste `index.html` (renomeado para isso) e `_headers` juntos.

**Automático (recomendado, via este repositório):** conecte este repositório na Netlify (New site from Git) e todo `git push` publica sozinho, sem esquecer nenhum arquivo.

## Stack

HTML + CSS + JavaScript puro, sem framework nem build. Bibliotecas via CDN: PDF.js, Mammoth.js, JSZip, Tesseract.js, Transformers.js. IA: Google Gemini API.

## Licença

Todos os direitos reservados. Código disponível para consulta e portfólio, não para reuso ou redistribuição — ver [`LICENSE`](LICENSE).
