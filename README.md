# queimadasR: Análise de Dados de Queimadas com Aplicações em Saúde Ambiental

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![R Version](https://img.shields.io/badge/R-%3E%3D4.0-blue.svg)](https://www.r-project.org/)
[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.18879882-blue.svg)](https://doi.org/10.5281/zenodo.18879882)
[![queimadasR](https://img.shields.io/badge/queimadasR-v0.2.0-brightgreen.svg)](https://github.com/wtassinari/queimadasR)

## 📖 Descrição

Este repositório contém os scripts e dados de suporte para o artigo **"queimadasR: pacote para download e análise de dados de queimadas do Instituto Nacional de Pesquisas Espaciais (INPE) com aplicações em Saúde Ambiental"**.

O projeto demonstra como integrar dados de focos de queimadas do INPE com dados de saúde do DATASUS, analisando a relação entre queimadas e internações hospitalares por causas respiratórias e circulatórias na Região Norte do Brasil (2020-2025).

## 🎯 Objetivos

O artigo e os scripts aqui apresentados têm como objetivos:

- **Apresentar o pacote `queimadasR`**: Uma ferramenta R para download automatizado de dados de queimadas do Instituto Nacional de Pesquisas Espaciais (INPE)
- **Integrar dados multifonte**: Combinar dados de queimadas, saúde e clima para análises abrangentes
- **Demonstrar aplicações em Saúde Ambiental**: Exemplificar como dados de queimadas podem ser relacionados com impactos na saúde pública
- **Fornecer código reprodutível**: Disponibilizar scripts documentados e reutilizáveis para a comunidade científica

## 📊 Figuras Principais

### Figura 1: Série Histórica Mensal (2020-2025)

Série histórica mensal dos focos de queimadas e das internações por causas respiratórias e circulatórias na Região Norte do Brasil (2020-2025).

**Características:**
- Barras vermelhas: Focos de queimadas (eixo y esquerdo)
- Linha azul: Internações respiratórias (eixo y direito)
- Linha verde: Internações circulatórias (eixo y direito)

### Figura 2: Distribuição Mensal (2022-2024)

Distribuição mensal da precipitação média, dias sem chuva (média) e FRP médio na data de ocorrência dos focos de queimadas na Região Norte do Brasil, 2022-2024.

**Características:**
- Barras azuis: Dias sem chuva (eixo y esquerdo)
- Linha verde: Precipitação média (eixo y esquerdo)
- Linha vermelha: FRP médio - Fire Radiative Power (eixo y direito)

## 🚀 Início Rápido

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
## Descrição: Script para reproduzir as análises e gráficos
##            apresentados no artigo, utilizando dados de focos
##            de queimadas da Região Norte do Brasil (2020-2025)
##            e dados de saúde do DATASUS
##
## Autor: [Seu Nome]
## Data: 2025
## Repositório: https://github.com/wtassinari/paper_queimadasR
##
## ============================================================


## ============================================================
## 0. SETUP INICIAL
## ============================================================

# Limpar ambiente
rm(list = ls())

# Carregar pacotes necessários
library(queimadasR)      # Download de dados de queimadas do INPE
library(microdatasus)    # Acesso a dados do DATASUS
library(dplyr)           # Manipulação de dados
library(tidyr)           # Transformação de dados
library(lubridate)       # Manipulação de datas
library(stringr)         # Manipulação de strings
library(purrr)           # Programação funcional (map_df)
library(ggplot2)         # Visualização de dados
library(gridExtra)       # Combinação de gráficos
library(scales)          # Escalas para ggplot2
library(tibble)          # Tibbles para dados estruturados

# Definir tema padrão para os gráficos
theme_set(theme_minimal() + 
          theme(panel.grid.major = element_line(color = "gray90"),
                panel.grid.minor = element_blank(),
                plot.title = element_text(size = 12, face = "bold"),
                axis.title = element_text(size = 10),
                legend.position = "bottom"))


## ============================================================
## 1. DICIONÁRIO DE CONVERSÃO: SIGLA UF → NOME COMPLETO
## ============================================================
## Necessário para harmonizar queimadasR (que usa nomes completos)
## com microdatasus (que usa siglas de UF)

uf_dict <- tibble::tribble(
  ~sigla,  ~nome_completo,
  "AC",    "ACRE",
  "AP",    "AMAPÁ"
)

# Vetores para uso posterior
siglas_norte <- uf_dict$sigla
nomes_norte  <- uf_dict$nome_completo


## ============================================================
## 2. AQUISIÇÃO DOS DADOS DE QUEIMADAS
## ============================================================
## Utiliza o pacote queimadasR para download de dados do INPE
## Período: 01/01/2020 a 31/12/2025
## Região: Estados da Região Norte (AC e AP)

cat("Iniciando download de dados de queimadas...\n")

focos <- download_fire_spots(
  start_date_str = "01/01/2020",
  end_date_str = "31/12/2025",
  target_states = nomes_norte,
  timeout = 1800,           # Recomendado: >= 1200 segundos
  deduplicate_final = TRUE  # Remove duplicatas
)

cat("Download concluído! Total de focos:", nrow(focos), "\n")

# Criar variáveis de agregação temporal
focos <- focos %>%
  mutate(
    ano    = format(as.Date(data_pas), "%Y"),
    mes    = format(as.Date(data_pas), "%m"),
    anomes = format(as.Date(data_pas), "%Y-%m"),
    estado = toupper(estado)
  )


## ============================================================
## 3. AQUISIÇÃO DOS DADOS DE INTERNAÇÕES (DATASUS)
## ============================================================
## Dados de internações por causas respiratórias e circulatórias
## Período: 2020-2025
## Região: Estados da Região Norte

cat("Iniciando download de dados de internações do DATASUS...\n")

# Função auxiliar para baixar dados de internações
baixar_internacoes <- function(sigla_uf, causa_cid) {
  tryCatch({
    dados <- fetch_datasus(
      year_start = 2020,
      year_end = 2025,
      uf = sigla_uf,
      information_system = "SIH"  # Sistema de Informações Hospitalares
    )
    
    # Filtrar por causa (CID-10)
    dados_filtrados <- dados %>%
      filter(str_detect(DIAG_PRINC, causa_cid)) %>%
      mutate(uf = sigla_uf)
    
    return(dados_filtrados)
  }, error = function(e) {
    cat("Erro ao baixar dados para", sigla_uf, ":", e$message, "\n")
    return(NULL)
  })
}

# Códigos CID-10 para as causas
# J00-J99: Doenças do aparelho respiratório
# I00-I99: Doenças do aparelho circulatório

internacoes_respiratorias <- map_df(
  siglas_norte,
  ~baixar_internacoes(.x, "^J")
)

internacoes_circulatorias <- map_df(
  siglas_norte,
  ~baixar_internacoes(.x, "^I")
)

cat("Download de internações concluído!\n")


## ============================================================
## 4. PROCESSAMENTO E AGREGAÇÃO DOS DADOS
## ============================================================

# Agregar focos por mês
focos_mes <- focos %>%
  group_by(anomes) %>%
  summarise(
    n_focos = n(),
    .groups = "drop"
  ) %>%
  mutate(
    data = as.Date(paste0(anomes, "-01"))
  ) %>%
  arrange(data)

# Agregar internações por mês
internacoes_mes <- bind_rows(
  internacoes_respiratorias %>% mutate(tipo = "Respiratória"),
  internacoes_circulatorias %>% mutate(tipo = "Circulatória")
) %>%
  mutate(
    data_internacao = as.Date(DT_INTER)  # Ajustar conforme nome da coluna
  ) %>%
  group_by(
    ano = year(data_internacao),
    mes = month(data_internacao),
    tipo
  ) %>%
  summarise(
    n_internacoes = n(),
    .groups = "drop"
  ) %>%
  mutate(
    data = as.Date(paste0(ano, "-", str_pad(mes, 2, pad = "0"), "-01"))
  ) %>%
  arrange(data)

# Pivotar internações para formato largo
internacoes_largo <- internacoes_mes %>%
  pivot_wider(
    names_from = tipo,
    values_from = n_internacoes,
    values_fill = 0
  )

# Juntar dados de focos e internações
dados_combinados <- focos_mes %>%
  left_join(internacoes_largo, by = "data") %>%
  replace_na(list(Respiratória = 0, Circulatória = 0)) %>%
  arrange(data)


## ============================================================
# Figure 1: Série histórica mensal dos focos de queimadas e das internações por causas respiratórias e circulatórias na Região Norte (2020-2025)
## ============================================================

# Preparar dados para o gráfico
dados_figura1 <- dados_combinados %>%
  filter(data >= as.Date("2020-01-01") & data <= as.Date("2025-12-31"))

# Criar gráfico com eixos duplos
fig1 <- ggplot(dados_figura1, aes(x = data)) +
  
  # Barras para focos de queimadas (eixo y esquerdo)
  geom_col(aes(y = n_focos), 
           fill = "#E74C3C", 
           alpha = 0.7,
           width = 20) +
  
  # Linha para internações respiratórias (eixo y direito)
  geom_line(aes(y = Respiratória * 100, color = "Internações respiratórias"),
            size = 1,
            na.rm = TRUE) +
  
  # Linha para internações circulatórias (eixo y direito)
  geom_line(aes(y = Circulatória * 100, color = "Internações circulatórias"),
            size = 1,
            na.rm = TRUE) +
  
  # Escala y esquerda (focos)
  scale_y_continuous(
    name = "Focos de queimadas",
    labels = label_comma(big.mark = ".", decimal.mark = ","),
    sec.axis = sec_axis(
      ~./100,
      name = "Internações por causas respiratória e circulatória",
      labels = label_comma(big.mark = ".", decimal.mark = ",")
    )
  ) +
  
  # Escala de cores para as linhas
  scale_color_manual(
    name = "",
    values = c(
      "Internações respiratórias" = "#3498DB",
      "Internações circulatórias" = "#2ECC71"
    )
  ) +
  
  # Formatação de datas no eixo x
  scale_x_date(
    date_breaks = "3 months",
    date_labels = "%b/%Y",
    expand = c(0, 0)
  ) +
  
  # Títulos e labels
  labs(
    title = "Série histórica mensal dos focos de queimadas e das internações\npor causas respiratórias e circulatórias na Região Norte do Brasil (2020-2025)",
    x = "Data",
    y = "Focos de queimadas"
  ) +
  
  # Tema
  theme(
    axis.text.x = element_text(angle = 45, hjust = 1, size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    plot.title = element_text(size = 11, face = "bold", hjust = 0),
    panel.grid.major.y = element_line(color = "gray90", linetype = "solid"),
    panel.grid.minor = element_blank()
  )

# Exibir figura 1
print(fig1)

# Salvar figura 1
ggsave("figura1_serie_historica.png", 
       fig1, 
       width = 14, 
       height = 7, 
       dpi = 300,
       bg = "white")

cat("Figura 1 salva como 'figura1_serie_historica.png'\n")


## ============================================================
## 5. AQUISIÇÃO DE DADOS CLIMÁTICOS E RADIAÇÃO
## ============================================================
## Dados de precipitação, dias sem chuva e FRP (Fire Radiative Power)
## para a data de ocorrência dos focos

# Extrair dados climáticos dos focos (já contêm informações de FRP)
dados_clima <- focos %>%
  mutate(
    mes_num = as.numeric(mes),
    mes_nome = month(as.Date(data_pas), label = TRUE, abbr = FALSE, locale = "pt_BR")
  ) %>%
  group_by(mes_num, mes_nome) %>%
  summarise(
    precipitacao_media = mean(precipitacao, na.rm = TRUE),
    dias_sem_chuva_media = mean(dias_sem_chuva, na.rm = TRUE),
    frp_media = mean(frp, na.rm = TRUE),
    n_focos = n(),
    .groups = "drop"
  ) %>%
  arrange(mes_num)

# Verificar nomes das colunas (ajustar conforme necessário)
cat("Colunas disponíveis nos dados de focos:\n")
print(names(focos))


## ============================================================
# Figure 2: Distribuição mensal da precipitação média, dias sem chuva (média) e FRP médio na data de ocorrência dos focos de queimadas na Região Norte do Brasil, 2022-2024
## ============================================================

# Preparar dados para o gráfico (2022-2024)
dados_figura2 <- focos %>%
  filter(ano %in% c("2022", "2023", "2024")) %>%
  mutate(
    mes_num = as.numeric(mes),
    mes_nome = factor(
      month(as.Date(data_pas), label = TRUE, abbr = TRUE, locale = "pt_BR"),
      levels = c("jan", "fev", "mar", "abr", "mai", "jun", 
                 "jul", "ago", "set", "out", "nov", "dez")
    )
  ) %>%
  group_by(mes_num, mes_nome) %>%
  summarise(
    precipitacao_media = mean(precipitacao, na.rm = TRUE),
    dias_sem_chuva_media = mean(dias_sem_chuva, na.rm = TRUE),
    frp_media = mean(frp, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  arrange(mes_num)

# Criar gráfico com eixos duplos
fig2 <- ggplot(dados_figura2, aes(x = mes_nome)) +
  
  # Barras para dias sem chuva (eixo y esquerdo)
  geom_col(aes(y = dias_sem_chuva_media),
           fill = "#5DADE2",
           alpha = 0.7,
           width = 0.6) +
  
  # Linha para precipitação (eixo y esquerdo)
  geom_line(aes(y = precipitacao_media, color = "Precipitação média"),
            size = 1.2,
            group = 1) +
  geom_point(aes(y = precipitacao_media, color = "Precipitação média"),
             size = 3) +
  
  # Linha para FRP (eixo y direito)
  geom_line(aes(y = frp_media / 5, color = "FRP médio"),
            size = 1.2,
            group = 1) +
  geom_point(aes(y = frp_media / 5, color = "FRP médio"),
             size = 3) +
  
  # Escala y esquerda (precipitação e dias sem chuva)
  scale_y_continuous(
    name = "Precipitação (mm) / Dias sem chuva (média)",
    sec.axis = sec_axis(
      ~. * 5,
      name = "FRP médio (W/m²)"
    )
  ) +
  
  # Escala de cores para as linhas
  scale_color_manual(
    name = "",
    values = c(
      "Precipitação média" = "#27AE60",
      "FRP médio" = "#E74C3C"
    )
  ) +
  
  # Títulos e labels
  labs(
    title = "Distribuição mensal da precipitação média, dias sem chuva (média)\ne FRP médio na data de ocorrência dos focos de queimadas\nna Região Norte do Brasil, 2022-2024",
    x = "Mês",
    y = "Precipitação (mm) / Dias sem chuva"
  ) +
  
  # Tema
  theme(
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    plot.title = element_text(size = 11, face = "bold", hjust = 0),
    panel.grid.major.y = element_line(color = "gray90", linetype = "solid"),
    panel.grid.minor = element_blank()
  )

# Exibir figura 2
print(fig2)

# Salvar figura 2
ggsave("figura2_distribuicao_mensal.png",
       fig2,
       width = 12,
       height = 7,
       dpi = 300,
       bg = "white")

cat("Figura 2 salva como 'figura2_distribuicao_mensal.png'\n")


## ============================================================
## 6. ESTATÍSTICAS DESCRITIVAS
## ============================================================

cat("\n========== ESTATÍSTICAS DESCRITIVAS ==========\n\n")

cat("Período de análise: 2020-2025\n")
cat("Região: Acre (AC) e Amapá (AP)\n\n")

cat("--- FOCOS DE QUEIMADAS ---\n")
cat("Total de focos:", nrow(focos), "\n")
cat("Focos por ano:\n")
print(focos %>% 
      group_by(ano) %>% 
      summarise(n_focos = n(), .groups = "drop"))

cat("\n--- PRECIPITAÇÃO ---\n")
cat("Precipitação média (mm):", mean(focos$precipitacao, na.rm = TRUE), "\n")
cat("Precipitação mínima (mm):", min(focos$precipitacao, na.rm = TRUE), "\n")
cat("Precipitação máxima (mm):", max(focos$precipitacao, na.rm = TRUE), "\n")

cat("\n--- DIAS SEM CHUVA ---\n")
cat("Dias sem chuva (média):", mean(focos$dias_sem_chuva, na.rm = TRUE), "\n")
cat("Dias sem chuva (mínimo):", min(focos$dias_sem_chuva, na.rm = TRUE), "\n")
cat("Dias sem chuva (máximo):", max(focos$dias_sem_chuva, na.rm = TRUE), "\n")

cat("\n--- FRP (FIRE RADIATIVE POWER) ---\n")
cat("FRP médio (W/m²):", mean(focos$frp, na.rm = TRUE), "\n")
cat("FRP mínimo (W/m²):", min(focos$frp, na.rm = TRUE), "\n")
cat("FRP máximo (W/m²):", max(focos$frp, na.rm = TRUE), "\n")


## ============================================================
## 7. EXPORTAÇÃO DE RESULTADOS
## ============================================================

# Salvar dados processados em CSV
write.csv(dados_combinados, 
          "dados_focos_internacoes.csv", 
          row.names = FALSE)

write.csv(dados_figura2, 
          "dados_clima_frp.csv", 
          row.names = FALSE)

cat("\n========== ANÁLISE CONCLUÍDA ==========\n")
cat("Arquivos gerados:\n")
cat("  - figura1_serie_historica.png\n")
cat("  - figura2_distribuicao_mensal.png\n")
cat("  - dados_focos_internacoes.csv\n")
cat("  - dados_clima_frp.csv\n")
```

## 📊 Dados Gerados

### Arquivos de Saída

O script gera automaticamente os seguintes arquivos:

#### Gráficos (PNG)
- `figura1_serie_historica.png` - Resolução 14×7 polegadas, 300 DPI
- `figura2_distribuicao_mensal.png` - Resolução 12×7 polegadas, 300 DPI

#### Dados Processados (CSV)
- `dados_focos_internacoes.csv` - Focos e internações agregados por mês
- `dados_clima_frp.csv` - Dados climáticos e FRP agregados por mês

## ⚠️ Notas Importantes

### Tempo de Execução
- O download de dados do INPE pode levar **15-30 minutos**
- O download de dados do DATASUS pode levar **10-20 minutos**
- Tempo total estimado: **30-50 minutos**

### Limitações de Dados
- O INPE pode ter atrasos na disponibilização de dados recentes
- Dados do DATASUS estão sujeitos a atualizações periódicas
- Alguns períodos podem ter dados incompletos ou indisponíveis
- A Figura 2 utiliza dados de 2022-2024 para melhor visualização

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

**Instalação:**
```r
remotes::install_github("wtassinari/queimadasR", force = TRUE)
```

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

