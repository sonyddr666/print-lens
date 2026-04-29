# RELATÓRIO TÉCNICO: print-lens

Este relatório documenta a investigação técnica feita durante o desenvolvimento do fluxo de automação para Google Lens com prints e imagens locais.

A conversa teve uma sequência bem caótica, mas tecnicamente produtiva: começamos tentando enviar uma imagem qualquer para o Lens, passamos por problemas de sessão, analisamos HARs, criamos múltiplos scripts, medimos tempos, otimizamos, quebramos OCR, batemos no bloqueio do Google e chegamos a uma arquitetura mais sensata.

## 1. Problema inicial

O objetivo inicial era:

```text
selecionar imagem local
enviar para Google Lens
obter resultado da IA/Google Lens
extrair texto/links no terminal
```

A expectativa era parecida com abrir manualmente uma imagem no Lens pelo Chrome e ver a tela com:

```text
Visão geral criada por IA
Correspondências visuais
Sobre esta imagem
Links relacionados
```

## 2. Primeira abordagem: URL já existente do Lens

No começo, foram testadas URLs completas do Google Lens/Search contendo parâmetros como:

```text
source=chrome.cr.tbic
cs=1
gsc=2
hl=pt-BR
lns_mode=un
lns_surface=42
gsessionid=...
udm=26
vsrid=...
vsint=...
```

A hipótese inicial era que bastava reproduzir essa URL.

Problema: a URL Lens depende de sessão e de imagem já associada à sessão do Google. Copiar a URL isolada não recria a sessão nem envia a imagem.

## 3. Upload com `requests` e problema de sessão

Foi criado um script que fazia:

```text
requests.post -> Google Lens/SearchByImage
Playwright -> abre URL de resultado
```

O upload retornava `302`/`303` com uma URL contendo `vsrid`, `vsint`, `gsessionid` e `lsessionid`.

Mas ao abrir no Playwright aparecia:

```text
A imagem não foi encontrada
A imagem que você está pesquisando não está associada à sua conta.
Faça um novo upload da imagem e tente mais uma vez.
```

Conclusão:

```text
upload em requests cria sessão A
Playwright abre com sessão B
Google rejeita imagem
```

Correção:

```text
usar context.request.post() do próprio Playwright
```

Assim upload e renderização compartilham cookies e contexto.

## 4. Primeiro fluxo funcional

O primeiro fluxo funcional foi:

```text
Playwright abre Chromium persistente
context.request.post envia imagem
Google retorna redirect com URL Lens
mesmo contexto abre a URL
página renderiza
script lê body/HTML
```

Isso resolveu o erro de imagem não associada.

Arquivos úteis gerados nas versões iniciais:

```text
lens_url.txt
lens_page_rendered.html
lens_text.txt
lens_resultado.json
```

## 5. Sessão persistente/interativa

Depois o objetivo mudou para não abrir e fechar o navegador por imagem.

Fluxo implementado:

```text
abre Chromium uma vez
prepara sessão Google
fica esperando input no terminal
usuário cola caminho da imagem
envia imagem
retorna resposta
volta a esperar
fecha só com sair/Ctrl+C
```

Isso reduziu overhead e manteve cookies vivos.

## 6. Automação por prints periódicos

Foi criado um modo que capturava a tela automaticamente com `mss`:

```text
captura tela
salva screenshot
envia ao Lens
renderiza resultado
mostra terminal
espera intervalo
repete
```

Testamos intervalos de:

- 15 segundos;
- 5 segundos;
- 4 segundos.

Com um motor, a análise renderizada levava aproximadamente:

```text
6 a 7 segundos
```

Depois, otimizando, alguns ciclos ficaram em torno de:

```text
2.5 a 3.5 segundos
```

Mas isso dependia do Google renderizar rápido e não bloquear.

## 7. Análise de HARs

Foram analisados HARs do Chrome/Google Lens.

### HAR com Lens verdadeiro

O fluxo real do Lens moderno mostrou:

```text
POST https://lens.google.com/v3/upload?ep=gsbubb&st=...&authuser=0&hl=pt-BR&vpw=...&vph=...
```

Esse request retornava `303` com `Location` apontando para:

```text
https://www.google.com/search?vsrid=...&vsint=...&udm=26&lns_mode=un&source=lns.web.gsbubb&gsessionid=...&lsessionid=...
```

### HAR que parecia Lens mas era busca normal

Outro HAR tinha:

```text
q=abelha
source=chrome.cr.tbic
```

Mas não tinha:

```text
udm=26
vsrid
vsint
gsessionid
lens.google.com/v3/upload
```

Conclusão:

```text
source=chrome.cr.tbic sozinho não prova Lens.
udm=26 + vsrid + vsint + gsessionid são marcadores melhores.
```

## 8. Dual engine

Foram testados dois motores:

### Motor 1

```text
https://lens.google.com/v3/upload?ep=gsbubb...
```

### Motor 2

```text
https://www.google.com/searchbyimage/upload
```

O script passou a enviar a mesma imagem para os dois motores e mostrar resultados separados.

### Sequencial

```text
motor 1 -> espera
motor 2 -> espera
```

Tempo típico:

```text
11 a 13 segundos por ciclo
```

### Paralelo

```text
motor 1 ┐
        ├ em paralelo
motor 2 ┘
```

Tempo típico otimizado:

```text
2.5 a 3.5 segundos em alguns testes
```

Porém, rodar dual engine em frequência alta aumenta chance de bloqueio.

## 9. Tentativa de OCR com Google Lens

Foi tentado enviar uma frase como:

```text
leia exatamente o texto
```

usando parâmetros:

```text
q=leia+exatamente+o+texto
oq=leia+exatamente+o+texto
udm=24
lns_mode=mu
```

Isso deu errado.

### Por quê?

Porque `q` no Google é query de busca, não prompt.

O Google passou a pesquisar a frase na web e retornar resultados aleatórios, como:

- YouTube;
- Reddit;
- Hostinger;
- Microsoft;
- páginas sobre OCR.

Conclusão:

```text
q=<texto> não é instrução para o Lens.
É consulta de busca.
```

## 10. Tentativa de clicar em "Selecionar texto"

Foi tentado:

```text
abrir Lens normal
clicar em Selecionar texto
clicar em Copiar texto
ler clipboard
```

Problema: o Google Lens Web não expôs de forma confiável o OCR limpo. O script acabava pegando `body.inner_text()` com a página inteira, misturando:

- Visão geral criada por IA;
- resultados visuais;
- links;
- rodapé;
- sugestões;
- referências.

Conclusão:

```text
Lens Web não é bom como OCR limpo automatizado.
```

A recomendação técnica foi separar:

```text
OCR local para texto exato
Google Lens para contexto visual e referências
```

## 11. Bloqueio do Google

Ao rodar o fluxo automático com resultado renderizado a cada 4 segundos, o Google retornou:

```text
Nossos sistemas detectaram tráfego incomum na sua rede de computadores.
Esta página verifica se é realmente você, e não um robô...
```

Isso apareceu depois de poucos ciclos.

Interpretação:

```text
abrir /search do Google Lens em loop rápido parece automação/bot
```

Importante:

```text
upload puro continuava rápido
renderizar resultado repetidamente causava bloqueio
```

## 12. Upload-only rápido

Foi criado um script que faz apenas:

```text
print 1280px
POST para Lens
captura URL do resultado
imprime tempos
```

Sem abrir/renderizar a URL.

Tempo típico:

```text
0.3 a 0.7 segundos por ciclo
```

Funcionou bem tecnicamente, mas não traz resultado textual no terminal, só a URL.

Conclusão:

```text
upload-only é rápido, mas não resolve sozinho se o usuário quer resultado.
```

## 13. Resultado rápido com render curto

Foi criado um meio-termo:

```text
print
upload
abre resultado por 2.5 a 4s
extrai o que tiver
```

Funcionou uma vez, mas logo acionou bloqueio do Google em loop automático.

Conclusão:

```text
resultado renderizado + loop 4s = não sustentável
```

## 14. Solução mais sensata: clipboard/manual

A solução final proposta foi mudar o gatilho:

```text
usuário dá PrintScreen ou Win+Shift+S
script detecta imagem nova no clipboard
envia ao Lens
abre resultado
mostra terminal
```

Vantagens:

- não martela Google continuamente;
- só envia quando o usuário realmente captura algo;
- parece mais uso humano;
- reduz risco de bloqueio;
- mantém automação útil.

## 15. Conclusões finais

### O que deu certo

- Enviar imagem ao Google Lens por Playwright.
- Usar endpoint `lens.google.com/v3/upload`.
- Manter sessão/cookies no mesmo contexto.
- Gerar URLs Lens válidas com `vsrid`, `vsint`, `gsessionid`.
- Upload rápido de prints.
- Detectar que renderizar resultado em loop agressivo causa bloqueio.
- Definir arquitetura manual via clipboard como mais viável.

### O que não deu certo

- Usar `requests` separado do Playwright.
- Usar URL Lens copiada sem recriar sessão.
- Usar `q=` como prompt.
- Fazer OCR limpo via Google Lens Web.
- Abrir/renderizar resultado a cada 4 segundos indefinidamente.
- Dual engine agressivo em loop.

### Melhor arquitetura final

```text
clipboard/manual trigger
  -> upload Lens
  -> renderiza resultado uma vez
  -> imprime resultado
```

### Para OCR exato

Não usar Google Lens Web como fonte principal. Usar OCR local.

### Para busca visual/contexto

Usar Google Lens.

## 16. Lições técnicas

1. Google Lens não é API pública estável.
2. Sessão é tudo: upload e render precisam do mesmo contexto/cookies.
3. `q=` é busca, não prompt.
4. Renderizar `/search` repetidamente é o que aciona bloqueio.
5. O endpoint `/v3/upload` é rápido e útil para criar sessão.
6. O modo manual reduz comportamento de bot.
7. OCR e Lens devem ser tratados como ferramentas diferentes.

## 17. Recomendação para continuidade

Implementar no repositório os scripts em camadas:

```text
1. lens_upload.py
   Apenas upload e URL.

2. lens_result.py
   Upload + render único.

3. lens_clipboard_manual.py
   Detecta PrintScreen/clipboard e processa.

4. ocr_local.py opcional
   OCR local separado, se o objetivo for texto exato.
```

E documentar claramente:

```text
não rodar render automático a cada 4s
usar modo manual para evitar bloqueio
```

## 18. Estado final do projeto

O projeto chegou em uma conclusão prática:

```text
Google Lens automatizado funciona para upload e resultado ocasional.
Não funciona bem como OCR contínuo nem como render automático agressivo.
```

A versão recomendada para uso real é baseada em clipboard/manual, não em loop automático.
