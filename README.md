# print-lens

Automação experimental para enviar capturas de tela ao Google Lens e obter uma URL/resultado visual usando Playwright.

> Status do projeto: pesquisa e protótipo. A parte de upload para o Google Lens funcionou bem. A extração automática do resultado renderizado funcionou em alguns cenários, mas ficou sujeita a bloqueio de tráfego incomum pelo Google quando executada em loop rápido.

## Objetivo

O objetivo inicial era simples:

1. escolher ou capturar uma imagem;
2. enviar essa imagem para o Google Lens;
3. receber de volta o resultado do Lens;
4. automatizar isso para prints de tela.

Durante os testes, o projeto evoluiu para vários modos:

- upload de imagem local;
- sessão persistente do Google Lens;
- processamento contínuo de imagens;
- captura automática de tela;
- dual engine usando dois endpoints Lens;
- tentativa de OCR via Google Lens;
- modo final mais seguro: detectar print manual no clipboard e enviar ao Lens.

## O que funcionou melhor

O fluxo mais estável tecnicamente foi:

```text
imagem/print
  -> POST para https://lens.google.com/v3/upload
  -> resposta 303 com Location
  -> URL Google Search/Lens com vsrid, vsint, gsessionid, lsessionid
```

Exemplo de URL gerada:

```text
https://www.google.com/search?vsrid=...&vsint=...&udm=26&lns_mode=un&source=lns.web.gsbubb&gsessionid=...&lsessionid=...
```

O upload em si costuma ser rápido, geralmente abaixo de 1 segundo.

## O que deu problema

Abrir a URL do resultado automaticamente em loop rápido, por exemplo a cada 4 ou 5 segundos, fez o Google responder com página de proteção:

```text
Nossos sistemas detectaram tráfego incomum na sua rede de computadores.
Esta página verifica se é realmente você, e não um robô...
```

Então a conclusão foi:

```text
upload frequente = geralmente OK
renderizar resultado /search em loop rápido = risco alto de bloqueio
```

## Requisitos

Python 3.10+ recomendado.

Instalação básica:

```bash
pip install playwright pillow beautifulsoup4 lxml mss
python -m playwright install chromium
```

Dependendo do script usado, nem todas as dependências são necessárias:

- `playwright`: sessão Chromium e POST no contexto do navegador;
- `pillow`: manipular imagem e clipboard;
- `mss`: capturar tela automaticamente;
- `beautifulsoup4` e `lxml`: extrair links/texto do HTML quando a página é renderizada.

## Principais descobertas técnicas

### 1. A sessão precisa ser a mesma

No começo, o upload era feito com `requests.post()` e depois o resultado era aberto com Playwright. Isso gerou erro:

```text
A imagem não foi encontrada.
A imagem que você está pesquisando não está associada à sua conta.
```

Causa:

```text
requests = sessão/cookies A
Playwright = sessão/cookies B
```

Correção:

```text
usar context.request.post() do próprio Playwright
```

Assim o upload e a leitura usam o mesmo contexto/cookies.

### 2. O endpoint moderno do Lens funcionou melhor

Endpoint testado e usado:

```text
https://lens.google.com/v3/upload?ep=gsbubb&st=<timestamp>&authuser=0&hl=pt-BR&vpw=1280&vph=1000
```

Ele responde com `303` e header `Location`, apontando para a URL de resultado.

### 3. Parâmetros importantes da URL Lens

Parâmetros úteis encontrados durante os testes:

| Parâmetro | Função observada |
|---|---|
| `vsrid` | Identificador da sessão/resultado visual. |
| `vsint` | Estado interno visual do Lens. |
| `gsessionid` | Sessão Google/Lens. |
| `lsessionid` | Sessão Lens adicional. |
| `udm=26` | Modo visual/Lens normal. |
| `lns_mode=un` | Modo Lens visual sem texto de query. |
| `source=lns.web.gsbubb` | Origem do fluxo Lens web/upload moderno. |
| `vsdim=1280,720` | Dimensão da imagem enviada. |

### 4. `q=` não é prompt, é busca

Tentamos enviar instruções como:

```text
leia exatamente o texto
```

pela URL usando `q=`/`oq=`, com `udm=24` e `lns_mode=mu`.

Isso não fez OCR. O Google tratou a frase como consulta de busca textual/multimodal e retornou links relacionados, como YouTube, Reddit e páginas aleatórias.

Conclusão:

```text
q=<texto> no Google Lens não é prompt de IA. É query de busca.
```

### 5. OCR via Google Lens Web não ficou confiável

Tentamos o caminho:

```text
abrir Lens normal
clicar em "Selecionar texto"
clicar em "Copiar texto"
ler clipboard
```

Mas na prática o DOM/clipboard retornou muitas vezes o texto da página inteira do Google Lens, misturando:

- Visão geral criada por IA;
- resultados visuais;
- links do Google;
- referências;
- rodapé;
- títulos de páginas.

Conclusão técnica:

```text
Google Lens Web é bom para contexto visual e busca por imagem, mas ruim como OCR automatizado limpo.
```

Para OCR exato de tela, a recomendação foi separar:

```text
OCR local -> texto exato
Google Lens -> contexto visual/referências
```

Este repositório, porém, continua focado em Google Lens.

## Modos testados

### Upload de uma imagem local

Primeiro fluxo:

```bash
python lens-config-ok.py --image asasd.png
```

Funcionou quando o upload passou a ser feito pelo contexto Playwright.

### Sessão persistente interativa

Fluxo desejado:

```text
abre Chromium uma vez
mantém sessão viva
espera caminho da imagem
envia imagem
mostra resultado
volta a esperar
```

Isso evita abrir/fechar navegador por imagem.

### Captura automática de tela

Testamos captura a cada 15s, depois 5s e 4s.

Com renderização de resultado, o Google começou a bloquear por tráfego incomum.

### Dual engine

Foram testados dois motores:

1. `lens.google.com/v3/upload`
2. `www.google.com/searchbyimage/upload`

Rodar os dois em sequência era lento. Rodar em paralelo melhorou muito o tempo, mas aumentou a pressão sobre o Google.

Conclusão:

```text
dual engine é útil para comparar endpoints, mas não é ideal para loop agressivo.
```

### Upload-only rápido

Fluxo:

```text
print -> upload -> URL
```

Sem renderizar resultado. Muito rápido, geralmente abaixo de 1 segundo.

Problema: não traz resultado no terminal, apenas a URL da sessão Lens.

### Resultado rápido

Fluxo:

```text
print -> upload -> abre URL por poucos segundos -> extrai texto disponível
```

Funcionou uma vez, mas em loop rápido levou a bloqueio do Google.

### Clipboard/manual

Fluxo recomendado no final:

```text
usuário aperta PrintScreen ou Win+Shift+S
script detecta imagem nova no clipboard
envia ao Google Lens
abre resultado
mostra no terminal
```

Esse modo reduz comportamento de bot, porque só envia quando o usuário realmente cria um print.

## Arquitetura recomendada

Para evitar bloqueios e manter utilidade:

```text
modo manual via clipboard
  -> detecta imagem nova
  -> envia ao Lens
  -> abre resultado uma vez
  -> imprime resumo
```

Não recomendado:

```text
loop automático a cada 4s abrindo resultado do Google
```

Motivo: risco alto de bloqueio por tráfego incomum.

## Exemplo de fluxo recomendado

```bash
python lens_clipboard_manual.py
```

Depois:

```text
PrintScreen
ou
Win + Shift + S
```

O script detecta o novo conteúdo do clipboard e processa.

## Limitações

- O Google pode bloquear automações de busca/renderização.
- O endpoint do Lens não é uma API pública documentada.
- Parâmetros como `vsrid`, `vsint`, `gsessionid` e `lsessionid` são internos e podem mudar.
- Renderizar resultado em alta frequência parece comportamento automatizado e gera bloqueio.
- OCR exato via Lens Web não foi confiável.
- O resultado textual depende do que o Google renderiza na página.

## Recomendações práticas

Para testes leves:

```text
usar modo manual via clipboard
aguardar entre envios
não abrir resultado em loop agressivo
não usar q/oq como prompt
```

Para OCR exato:

```text
usar OCR local separado
```

Para contexto visual/referências:

```text
usar Google Lens
```

## Estrutura sugerida do projeto

```text
print-lens/
  README.md
  RELATORIO.md
  lens_clipboard_manual.py
  lens_google_upload_4s_fast.py
  lens_google_4s_resultado_rapido.py
  experiments/
    antigos scripts de teste
```

## Status final

O projeto confirmou que é possível automatizar o upload de prints para o Google Lens e obter URLs de resultado com sessão válida.

A extração automática de resultado no terminal é possível, mas deve ser usada com cuidado. Em loop rápido, o Google bloqueia.

A melhor experiência encontrada foi a automação manual baseada no clipboard: o usuário dá print, o script detecta e envia.
