# Decisões (estilo ADR)

**Português** · [English](decisions.md)

Registros curtos das decisões que moldaram o mapa público. Cada uma se baseia no código de produção; o código em si não é publicado.

## 1. O LLM pode vetar, nunca aprovar por conta própria

**Contexto.** Os reportes diários de posição são digitados por pessoas no mar. Erros de digitação acontecem (um dígito faltando, um hemisfério trocado), e alguns erros passam em verificações numéricas simples mesmo sendo implausíveis no contexto. Um LLM consegue julgar o contexto, mas também pode errar ou ficar indisponível.

**Decisão.** Regras rígidas rodam primeiro (mais recente que o publicado, dentro de uma bounding box regional, velocidade implícita de no máximo 15 nós). Apenas os reportes aprovados são enviados ao modelo, com o histórico recente, o período da operação e o polígono da área de trabalho. O modelo responde em um schema JSON estrito; ele pode transformar uma aprovação em retenção, nunca o contrário. O texto de posição na notificação é sempre gerado de forma determinística; o modelo apenas acrescenta um comentário.

**Consequências.** O pior caso de um erro do modelo é um reporte retido que uma pessoa libera, e não uma posição errada em um mapa público. Respostas que não podem ser interpretadas ou que estão fora do schema são tratadas como "revisor indisponível".

## 2. Fail-open no revisor, fail-closed nas regras

**Contexto.** O revisor depende de uma chave de API externa e de uma chamada de rede. Se ele cair, o mapa pode parar de se atualizar.

**Decisão.** Quando o revisor estiver indisponível, publicar apenas com base nas regras e dizer isso na notificação enviada ao operador. Quando uma regra falhar, nunca publicar.

**Consequências.** O mapa continua se atualizando durante uma indisponibilidade da API, e o operador consegue ver exatamente quais publicações ficaram sem segunda opinião. A troca é aceita porque as regras já limitam o erro plausível.

## 3. Sem banco de dados na aplicação do mapa

**Contexto.** As operações já são gerenciadas, revisadas e publicadas no portal de conformidade. Um segundo banco significaria sincronização e uma segunda fonte da verdade.

**Decisão.** Ler as operações no servidor a partir do endpoint público do portal, com cache de 60 segundos. Armazenar a última posição reportada e um histórico de dez entradas em um único arquivo JSON no armazenamento persistente do App Service. Serializar as ingestões no processo.

**Consequências.** Atualizações de conteúdo nunca exigem rebuild ou redeploy, e o app é barato de manter no ar. Um único arquivo JSON e um lock no processo pressupõem uma única instância; escalar horizontalmente exigiria um armazenamento compartilhado.

## 4. Terreno próprio em WebGL2 em vez do terreno raster-DEM nativo

**Contexto.** A primeira visualização 3D usava o terreno raster-DEM do MapLibre, construído a partir da mesma batimetria. O tempo de quadro e as long tasks na thread principal durante a navegação eram o gargalo.

**Decisão.** Escrever uma custom layer do MapLibre em WebGL2 puro: LOD em quadtree, patches gerados no vertex shader sem vertex buffers, uma única função de altura compartilhada para evitar fissuras, sombreamento e isóbatas por pixel, um controlador de qualidade e recuperação de perda de contexto. Gerar os tiles de altura offline, a partir de uma grade pública do GEBCO, como WebP sem perdas com metros inteiros exatos.

**Consequências.** No A/B registrado nas notas do projeto: cerca de 2,7x menos tempo de quadro, nenhuma long task durante a navegação, aproximadamente um quarto do heap de JavaScript. O custo é manter código de shader e de geometria que, de outra forma, uma biblioteca manteria, o que é atenuado por testes que amarram o mapeamento de tiles do cliente aos metadados do gerador.

## 5. Público por padrão, autenticação mantida como chave em tempo de execução

**Contexto.** A primeira versão exigia login. Os usuários pretendidos (pescadores) não deveriam precisar de conta para ver onde uma pesquisa está trabalhando.

**Decisão.** Abrir a interface para todos, mas manter o código de autenticação (hash scrypt, cookie assinado com HMAC) intacto e testado, ativado por uma app setting lida em tempo de execução.

**Consequências.** O produto pode voltar a um modo com acesso restrito com um restart, não com um rebuild. Abrir a interface não ampliou a exposição de dados: os dados já estavam limitados à projeção pública do portal, protegida pelo duplo gate.
