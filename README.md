# queimadasR: Análise de Dados de Queimadas com Aplicações em Saúde Ambiental

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![R Version](https://img.shields.io/badge/R-%3E%3D4.0-blue.svg)](https://www.r-project.org/)
[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.18879882-blue.svg)](https://doi.org/10.5281/zenodo.18879882)
[![queimadasR](https://img.shields.io/badge/queimadasR-v0.2.0-brightgreen.svg)](https://github.com/wtassinari/queimadasR)

## 📖 Descrição

Este repositório contém os scripts e dados de suporte para o artigo **"queimadasR: pacote para download e análise de dados de queimadas do Instituto Nacional de Pesquisas Espaciais (INPE) com aplicações em Saúde Ambiental"**.

O projeto demonstra como integrar dados de focos de queimadas do INPE com dados de saúde do DATASUS, analisando a relação entre queimadas e internações hospitalares por causas respiratórias e circulatórias de algumas UFs da Região Norte do Brasil (2020-2025).

## 🎯 Objetivos

O artigo e os scripts aqui apresentados têm como objetivos:

- **Apresentar o pacote `queimadasR`**: Uma ferramenta R para download automatizado de dados de queimadas do Instituto Nacional de Pesquisas Espaciais (INPE)
- **Integrar dados multifonte**: Combinar dados de queimadas, saúde e clima para análises abrangentes
- **Demonstrar aplicações em Saúde Ambiental**: Exemplificar como dados de queimadas podem ser relacionados com impactos na saúde pública
- **Fornecer código reprodutível**: Disponibilizar scripts documentados e reutilizáveis para a comunidade científica

## 📊 Figuras Principais

### Figura 1: Série Histórica Mensal (2020-2025)

Série histórica mensal dos focos de queimadas e das internações por causas respiratórias e circulatórias de algumas UFs da Região Norte do Brasil (2020-2025).

**Características:**
- Barras vermelhas: Focos de queimadas (eixo y esquerdo)
- Linha azul: Internações respiratórias (eixo y direito)
- Linha verde: Internações circulatórias (eixo y direito)

### Figura 2: Distribuição Mensal (2020-2025)

Distribuição mensal da precipitação média e FRP médio na data de ocorrência dos focos de queimadas  de algumas UFs da Região Norte do Brasil, 2022-2024.

**Características:**

- Linha verde: Precipitação média (eixo y esquerdo)
- Linha vermelha: FRP médio - Fire Radiative Power (eixo y direito)

### Pré-requisitos

- **R** versão 4.0 ou superior
- **RStudio** (recomendado, mas não obrigatório)
- Conexão com a internet (para download de dados)

### Instalação de Pacotes

Execute o seguinte comando no R para instalar todos os pacotes necessários:

```r
# Instalar pacotes do CRAN
install.packages(c(
  "dplyr", "tidyr", "lubridate", "stringr", "purrr",
  "ggplot2", "gridExtra", "scales", "tibble"
))

# Instalar pacotes do GitHub
if (!require("remotes")) install.packages("remotes")
remotes::install_github("wtassinari/queimadasR", force = TRUE)
remotes::install_github("rfsaldanha/microdatasus")
```

### Execução do Script

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/wtassinari/paper_queimadasR.git
   cd paper_queimadasR
   ```

2. **Abra o script no RStudio:**
   ```r
   # No console do R
   source("script_figuras_queimadasR.R")
   ```

3. **Aguarde a conclusão:**
   - O download de dados pode levar 15-30 minutos
   - Os gráficos serão salvos automaticamente em PNG
   - Os dados processados serão salvos em CSV

## 💻 Script Completo

O script abaixo reproduz as análises e gráficos apresentados no artigo. Ele está totalmente documentado e comentado para facilitar a compreensão e modificação.

```r
## ============================================================
## SCRIPT: Reprodução das Figuras do Artigo queimadasR
## ============================================================
##
## Título: queimadasR: pacote para download e análise de dados
##         de queimadas do Instituto Nacional de Pesquisas
##         Espaciais (INPE) com aplicações em Saúde Ambiental
##
## Descrição: Reproduz as figuras do artigo a partir de dados de
##            focos de queimadas (INPE, via pacote queimadasR) e
##            de internações hospitalares (DATASUS/SIH, via
##            pacote microdatasus) para os estados da Região
##            Norte do Brasil, 2020-2025.
##
## Repositório: https://github.com/wtassinari/paper_queimadasR
##
## NOTA: Ajuste `siglas_norte` na Seção 1 caso queira trabalhar
##       com um subconjunto de estados diferente do usado aqui.
## ============================================================


## ============================================================
## 0. SETUP INICIAL
## ============================================================

rm(list = ls())

library(queimadasR)      # Download de dados de queimadas do INPE
library(microdatasus)    # Acesso a dados do DATASUS
library(dplyr)           # Manipulação de dados
library(tidyr)           # Transformação de dados
library(lubridate)       # Manipulação de datas
library(stringr)         # Manipulação de strings
library(purrr)           # Programação funcional (map_df, map2_df)
library(ggplot2)         # Visualização de dados
library(scales)          # Escalas para ggplot2
library(tibble)          # Tibbles para dados estruturados


## ============================================================
## 1. ESTADOS DA REGIÃO NORTE
## ============================================================
## Dicionário sigla -> nome completo, necessário porque o
## queimadasR usa nomes completos de UF e o microdatasus usa
## siglas.

uf_dict <- tribble(
  ~sigla, ~nome_completo,
  "AC",   "ACRE",
  "AP",   "AMAPÁ",
  "RO",   "RONDÔNIA"
)

siglas_norte <- uf_dict$sigla
nomes_norte  <- uf_dict$nome_completo


## ============================================================
## ETAPA 1 — Aquisição dos dados de queimadas (queimadasR)
## ============================================================

focos <- download_fire_spots(
  start_date_str     = "01/01/2020",
  end_date_str       = "31/12/2025",
  target_states      = nomes_norte,
  target_satellites  = c("AQUA_M-T"),   # satélite de referência do INPE
  timeout            = 1800,            # recomendado >= 1200s
  deduplicate_final  = TRUE
)

## Criar variáveis de agregação temporal
focos <- focos %>%
  mutate(
    ano    = format(as.Date(data_pas), "%Y"),
    mes    = format(as.Date(data_pas), "%m"),
    anomes = format(as.Date(data_pas), "%Y-%m"),
    estado = toupper(estado)   # garantir maiúsculas (já vem assim, mas por segurança)
  )


## ============================================================
## ETAPA 2 — Agregação mensal dos indicadores ambientais
## ============================================================

indicadores_amb <- focos %>%
  group_by(estado, anomes, ano, mes) %>%
  summarise(
    n_focos            = n(),
    frp_medio          = mean(frp, na.rm = TRUE),
    frp_total          = sum(frp, na.rm = TRUE),
    precipitacao_media = mean(precipitacao, na.rm = TRUE),
    dias_sem_chuva_med = mean(numero_dias_sem_chuva, na.rm = TRUE),
    risco_fogo_medio   = mean(risco_fogo, na.rm = TRUE),
    .groups = "drop"
  )


## ============================================================
## 3. AQUISIÇÃO DOS DADOS DE INTERNAÇÕES (DATASUS/SIH)
## ============================================================
## CID-10:
##   J00-J99 -> Doenças do aparelho respiratório
##   I00-I99 -> Doenças do aparelho circulatório
## Período: 2020-2025

sih_norte <- fetch_datasus(
  year_start         = 2020,
  month_start        = 1,
  year_end           = 2025,
  month_end          = 12,
  uf                 = siglas_norte,
  information_system = "SIH-RD"
)

## Pré-processamento das variáveis do SIH
sih_norte <- process_sih(sih_norte)

## Garantir que DT_INTER seja character antes de converter
## (process_sih pode alterar o tipo da coluna)
sih_norte <- sih_norte %>%
  mutate(DT_INTER = as.character(DT_INTER))

## Filtragem por capítulos da CID-10
sih_resp <- sih_norte %>%
  filter(grepl("^J", DIAG_PRINC, ignore.case = FALSE))   # Respiratório (J00–J99)

sih_circ <- sih_norte %>%
  filter(grepl("^I", DIAG_PRINC, ignore.case = FALSE))   # Circulatório (I00–I99)

## Junta as duas causas num único data frame, marcando o tipo,
## para manter compatibilidade com a Etapa 4 (agregação mensal)
internacoes <- bind_rows(
  sih_resp %>% mutate(tipo = "Respiratória"),
  sih_circ %>% mutate(tipo = "Circulatória")
)

## ============================================================
## 4. PROCESSAMENTO E AGREGAÇÃO DOS DADOS
## ============================================================

# Focos agregados por mês (todos os estados combinados)
focos_mes <- focos %>%
  mutate(anomes = format(as.Date(data_pas), "%Y-%m")) %>%
  group_by(anomes) %>%
  summarise(n_focos = n(), .groups = "drop") %>%
  mutate(data = as.Date(paste0(anomes, "-01"))) %>%
  arrange(data)

# Internações agregadas por mês e tipo de causa
internacoes_mes <- internacoes %>%
  # A coluna DT_INTER já está no formato "YYYY-MM-DD", 
  # então basta usar as.Date() diretamente ou ymd() do lubridate
  mutate(data_internacao = as.Date(DT_INTER)) %>%
  # Remover registros onde a data é NA para evitar erros nas etapas seguintes
  drop_na(data_internacao) %>%
  # Agrupar por ano, mês e tipo
  group_by(
    ano = year(data_internacao),
    mes = month(data_internacao),
    tipo
  ) %>%
  # Contar o número de internações
  summarise(n_internacoes = n(), .groups = "drop") %>%
  
  # Recriar a data para o primeiro dia do mês
  mutate(
    # Usando make_date do lubridate (mais seguro e limpo que paste0 + as.Date)
    data = make_date(year = ano, month = mes, day = 1)
  ) %>%
  arrange(data)

# Formato longo: uma coluna por tipo de causa
internacoes_longo <- internacoes_mes %>%
  pivot_wider(
    names_from  = tipo,
    values_from = n_internacoes,
    values_fill = 0
  )

# Junta focos de queimadas e internações num único data frame mensal
dados_combinados <- focos_mes %>%
  left_join(internacoes_longo, by = "data") %>%
  replace_na(list(Respiratória = 0, Circulatória = 0)) %>%
  arrange(data)

## ============================================================
## FIGURA 1: Série histórica mensal — focos de queimadas x
## internações por causas respiratórias e circulatórias
## (Região Norte, 2020-2025)
## ============================================================
dados_figura1 <- dados_combinados %>%
  filter(data >= as.Date("2020-01-01"), data <= as.Date("2025-12-31"))

# Defina o teto desejado para cada eixo
limite_focos       <- 50000
limite_internacoes <- 6000  # ajuste conforme o valor máximo real de internações

# Fator de conversão entre as duas escalas
fator <- limite_focos / limite_internacoes

fig1 <- ggplot(dados_figura1, aes(x = data)) +
  
  # Coluna de focos de queimadas (escala primária, sem transformação)
  geom_col(aes(y = n_focos), fill = "#E74C3C", alpha = 0.7, width = 20) +
  
  # Linhas de internações — multiplicadas pelo fator para "subir" até a escala do eixo primário
  geom_line(aes(y = Respiratória * fator, color = "Internações respiratórias"),
            linewidth = 1, na.rm = TRUE) +
  geom_line(aes(y = Circulatória * fator, color = "Internações circulatórias"),
            linewidth = 1, na.rm = TRUE) +
  
  scale_y_continuous(
    name = "Focos de queimadas",
    limits = c(0, limite_focos),
    labels = label_comma(big.mark = ".", decimal.mark = ","),
    sec.axis = sec_axis(
      ~. / fator,
      name = "Internações por causas respiratória e circulatória",
      labels = label_comma(big.mark = ".", decimal.mark = ",")
    )
  ) +
  
  scale_color_manual(
    name = "",
    values = c(
      "Internações respiratórias" = "#3498DB",
      "Internações circulatórias" = "#2ECC71"
    )
  ) +
  
  scale_x_date(date_breaks = "3 months", date_labels = "%b/%Y", expand = c(0, 0)) +
  
  labs(
    x = "Data",
    y = "Focos de queimadas"
  ) +
  
  theme_minimal() +
  theme(
    axis.text.x         = element_text(angle = 45, hjust = 1, size = 9),
    axis.text.y          = element_text(size = 9),
    plot.title           = element_text(size = 11, face = "bold", hjust = 0),
    panel.grid.major.y   = element_line(color = "gray90"),
    panel.grid.minor     = element_blank(),
    panel.background     = element_rect(fill = "white", color = NA),
    plot.background       = element_rect(fill = "white", color = NA),
    legend.position       = "bottom",
    legend.title          = element_blank()
  )

print(fig1)

ggsave("figura1_serie_historica_mensal.png", fig1, width = 12, height = 7, dpi = 300, bg = "white")

## ============================================================
## FIGURA 2: Distribuição mensal — precipitação média, dias sem
## chuva (média) e FRP médio na data de ocorrência dos focos
## (Região Norte, 2020-2025)
## ============================================================
#

library(dplyr)
library(ggplot2)
library(scales)

# ------------------------------------------------------------
# 1. Limpeza: substituir -999 por NA em numero_dias_sem_chuva
# ------------------------------------------------------------
focos <- focos %>%
  mutate(numero_dias_sem_chuva = na_if(numero_dias_sem_chuva, -999))

# ------------------------------------------------------------
# 2. Agregação mensal
# ------------------------------------------------------------
meses_ordem <- c("jan", "fev", "mar", "abr", "mai", "jun",
                 "jul", "ago", "set", "out", "nov", "dez")

dados_figura2 <- focos %>%
  filter(ano %in% c("2020", "2021", "2022", "2023", "2024", "2025")) %>%
  mutate(mes_num = as.numeric(mes)) %>%   # garanta que 'mes' já é 1-12
  group_by(mes_num) %>%
  summarise(
    precipitacao_media   = mean(precipitacao, na.rm = TRUE),
    frp_media            = mean(frp, na.rm = TRUE),
    dias_sem_chuva_media = mean(numero_dias_sem_chuva, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  arrange(mes_num) %>%
  mutate(mes_nome = factor(meses_ordem[mes_num], levels = meses_ordem))

# ------------------------------------------------------------
# 3. Fatores de escala — agora tudo em função do FRP (base = coluna)
# ------------------------------------------------------------
fator_precip <- max(dados_figura2$frp_media, na.rm = TRUE) /
  max(dados_figura2$precipitacao_media, na.rm = TRUE)

fator_dias <- max(dados_figura2$frp_media, na.rm = TRUE) /
  max(dados_figura2$dias_sem_chuva_media, na.rm = TRUE)

# ------------------------------------------------------------
# 4. Gráfico
# ------------------------------------------------------------
fig2 <- ggplot(dados_figura2, aes(x = mes_nome)) +
  
  # Coluna de FRP médio (vermelha)
  geom_col(aes(y = frp_media),
           fill = "#E74C3C", alpha = 0.7, width = 0.6) +
  
  # Linha de precipitação média
  geom_line(aes(y = precipitacao_media * fator_precip, color = "Precipitação média"),
            linewidth = 1.2, group = 1) +
  geom_point(aes(y = precipitacao_media * fator_precip, color = "Precipitação média"), size = 3) +
  
  # Linha de dias sem chuva (média)
  geom_line(aes(y = dias_sem_chuva_media * fator_dias, color = "Dias sem chuva (média)"),
            linewidth = 1.2, group = 1) +
  geom_point(aes(y = dias_sem_chuva_media * fator_dias, color = "Dias sem chuva (média)"), size = 3) +
  
  # Escala Y com eixo secundário (referência: precipitação, à direita)
  scale_y_continuous(
    name = "FRP médio (W/m²)",
    labels = label_comma(big.mark = ".", decimal.mark = ","),
    sec.axis = sec_axis(~. / fator_precip,
                        name = "Precipitação média (mm)",
                        labels = label_comma(big.mark = ".", decimal.mark = ","))
  ) +
  
  # Cores das linhas
  scale_color_manual(
    name = "",
    values = c(
      "Precipitação média"     = "#5DADE2",
      "Dias sem chuva (média)" = "#23c423"
    )
  ) +
  
  # Labels
  labs(
    x = "Mês",
    y = NULL
  ) +
  
  # Tema
  theme_minimal() +
  theme(
    axis.text.x        = element_text(size = 10),
    axis.text.y        = element_text(size = 9),
    plot.title          = element_text(size = 11, face = "bold", hjust = 0),
    panel.grid.major.y  = element_line(color = "gray90"),
    panel.grid.minor    = element_blank(),
    legend.position     = "bottom",
    legend.title        = element_blank()
  )

print(fig2)

ggsave("figura2_distribuicao_mensal.png", fig2, width = 12, height = 7, dpi = 300, bg = "white")
```

## ⚠️ Notas Importantes

### Tempo de Execução
- O download de dados do INPE pode levar **15-30 minutos**
- O download de dados do DATASUS pode levar **10-20 minutos**
- Tempo total estimado: **30-50 minutos**

### Limitações de Dados
- O INPE pode ter atrasos na disponibilização de dados recentes
- Dados do DATASUS estão sujeitos a atualizações periódicas
- Alguns períodos podem ter dados incompletos ou indisponíveis

### Requisitos de Conexão
- Conexão estável com a internet é essencial
- O parâmetro `timeout = 1800` (30 minutos) pode precisar ser aumentado em conexões lentas

## 📚 Referências

### Pacote queimadasR

O pacote `queimadasR` permite acesso direto aos dados do sistema BDQueimadas, incluindo:

- 🔥 Focos de calor
- 🌡️ Índice de risco de fogo
- 🌧️ Variáveis meteorológicas associadas
- 🌎 Informações espaciais (estado, município, bioma)
- ⚡ FRP (Fire Radiative Power)

Os dados são oficiais e públicos, fornecidos pelo Instituto Nacional de Pesquisas Espaciais através do Programa Queimadas.

**Citação do pacote queimadasR:**

**Formato ABNT:**
TASSINARI, Wagner S.; PACIFICO, Roni dos Santos Jorge; FERREIRA, Manuela dos Santos; OLIVEIRA, Liliane de Fátima Antônio; HOKERBERG, Yara Hahr Marques; SANTOS, Heloísa Ferreira Pinto; SAUCHA, Camylla Veloso Valença; OLIVEIRA, Raquel de Vasconcellos Carvalhaes. **queimadasR**: Pacote para download e análise de dados de queimadas do INPE. Versão 0.2.0. 2026. Disponível em: https://github.com/wtassinari/queimadasR

**Formato BibTeX:**
```bibtex
@software{queimadasR2026,
  title = {queimadasR: Pacote para download e análise de dados de queimadas do INPE},
  author = {Tassinari, Wagner S. and Pacifico, Roni dos Santos Jorge and 
            Ferreira, Manuela dos Santos and Oliveira, Liliane de Fátima Antônio and 
            Hokerberg, Yara Hahr Marques and Santos, Heloísa Ferreira Pinto and 
            Saucha, Camylla Veloso Valença and Oliveira, Raquel de Vasconcellos Carvalhaes},
  organization = {Universidade Federal Rural do Rio de Janeiro e Instituto Nacional de Infectologia/FIOCRUZ},
  year = {2026},
  version = {0.2.0},
  doi = {10.5281/zenodo.18879882},
  url = {https://doi.org/10.5281/zenodo.18879882}
}
```

**Mais informações:** [Portal BDQueimadas INPE](https://terrabrasilis.dpi.inpe.br/queimadas/bdqueimadas/)

