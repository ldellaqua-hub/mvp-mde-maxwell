# mvp-mde-maxwell
Projeto para disciplina de Mineração de dados professor Maxwell
import streamlit as st
import pandas as pd

st.set_page_config(page_title="Painel de Rendimento - ES", layout="wide")

# ====== Cabeçalho ======
st.title("Painel de Taxas de Rendimento – Rede Estadual/ES (por Município)")
st.caption("por **Luciene Dellaqua Bergamin** • Fonte: INEP (Censo Escolar/Situação do Aluno)")

# ====== Navegação ======
sec = st.sidebar.radio(
    "Seções",
    ["Início", "Visão Geral – ES", "Ranking de Municípios", "Evolução Temporal", "Comparador", "Metodologia & Fontes"]
)

# ====== Upload de dados ======
st.sidebar.subheader("Carregar dados")
rend_file = st.sidebar.file_uploader("Taxas por município (ex.: tx_rend_municipios_2024.xlsx)", type=["xlsx", "csv"])
micro_file = st.sidebar.file_uploader("Microdados (opcional)", type=["csv", "zip"])

@st.cache_data
def load_table(file):
    if file is None:
        return None
    name = file.name.lower()
    if name.endswith(".xlsx"):
        return pd.read_excel(file)
    else:
        return pd.read_csv(file, sep=";", encoding="latin1").rename(columns=str.strip)

df_rend = load_table(rend_file)

# ====== Seções ======
if sec == "Início":
    st.markdown("""
    **Objetivo:** apresentar as taxas de **Aprovação, Reprovação e Abandono** da **rede estadual do ES**, por **município**.
    
    **Como usar:**
    - Carregue o arquivo de **taxas por município** na barra lateral.
    - Navegue pelas seções para ver panorama, rankings, séries históricas e comparações.
    
    **Escopo (versão 1):**
    - Recorte: rede **estadual**.
    - Unidade de análise: **município**.
    - Expansões futuras: **escolas dentro do município**, correlação com **infraestrutura** e **IDEB/SAEB**, **mapa**.
    """)
    st.info("Dica: padronize chaves com código IBGE ou INEP do município para garantir junções consistentes.")

elif sec == "Visão Geral – ES":
    if df_rend is None:
        st.warning("Carregue primeiro a tabela de taxas por município.")
    else:
        # Adapte os nomes das colunas conforme seu arquivo (ex.: MUNICIPIO, ANO, APROVACAO, REPROVACAO, ABANDONO)
        col_map = {c.lower(): c for c in df_rend.columns}
        aprox_cols = ["aprovação", "aprovacao", "tx_aprovacao"]
        reprov_cols = ["reprovação", "reprovacao", "tx_reprovacao"]
        aband_cols = ["abandono", "tx_abandono"]

        def pick(cols):
            for k in cols:
                if k in col_map: return col_map[k]
            return None

        c_apv = pick([c for c in col_map if any(a in c for a in aprox_cols)])
        c_rep = pick([c for c in col_map if any(a in c for a in reprov_cols)])
        c_abd = pick([c for c in col_map if any(a in c for a in aband_cols)])

        ano_col = col_map.get("ano") or col_map.get("year")
        muni_col = col_map.get("municipio") or col_map.get("município")

        if not all([c_apv, c_rep, c_abd, muni_col]):
            st.error("Não encontrei colunas esperadas (ano, município, aprovação, reprovação, abandono). Ajuste os nomes no arquivo.")
        else:
            if ano_col in df_rend.columns:
                anos = sorted(df_rend[ano_col].dropna().unique())
                ano = st.selectbox("Selecione o ano", anos, index=len(anos)-1 if len(anos)>0 else 0)
                base = df_rend[df_rend[ano_col]==ano].copy()
            else:
                base = df_rend.copy()

            # Médias estaduais simples (ponderações podem ser aplicadas futuramente)
            kpi1 = base[c_apv].mean()
            kpi2 = base[c_rep].mean()
            kpi3 = base[c_abd].mean()

            c1,c2,c3 = st.columns(3)
            c1.metric("Aprovação média (%)", f"{kpi1:0.1f}")
            c2.metric("Reprovação média (%)", f"{kpi2:0.1f}")
            c3.metric("Abandono médio (%)", f"{kpi3:0.1f}")

            st.subheader("Tabela resumida")
            st.dataframe(base[[muni_col, c_apv, c_rep, c_abd]].sort_values(c_apv, ascending=False), use_container_width=True)

elif sec == "Ranking de Municípios":
    if df_rend is None:
        st.warning("Carregue primeiro a tabela de taxas por município.")
    else:
        # Ajuste nomes conforme seu arquivo
        col_map = {c.lower(): c for c in df_rend.columns}
        ano_col = col_map.get("ano") or col_map.get("year")
        muni_col = col_map.get("municipio") or col_map.get("município")

        order_by = st.selectbox("Ordenar por", ["Aprovação", "Reprovação", "Abandono"])
        asc = st.toggle("Ordem crescente", value=False)

        if ano_col in df_rend.columns:
            anos = sorted(df_rend[ano_col].dropna().unique())
            ano = st.selectbox("Ano", anos, index=len(anos)-1 if len(anos)>0 else 0)
            base = df_rend[df_rend[ano_col]==ano].copy()
        else:
            base = df_rend.copy()

        col_key = [c for c in base.columns if order_by.lower() in c.lower()]
        if not col_key:
            st.error("Coluna do indicador não encontrada; verifique nomes no arquivo.")
        else:
            st.dataframe(
                base[[muni_col, col_key[0]]].sort_values(col_key[0], ascending=asc),
                use_container_width=True
            )

elif sec == "Evolução Temporal":
    st.info("Versão inicial: após padronizar colunas, adicionar gráfico de linhas com a série histórica por município selecionado.")

elif sec == "Comparador":
    st.info("Versão inicial: selecionar 2–3 municípios e comparar indicadores lado a lado.")

elif sec == "Metodologia & Fontes":
    st.markdown("""
    **Metodologia (INEP – Situação do Aluno/Censo Escolar):**  
    - *Taxa de Aprovação (%)* = Aprovados / (Aprovados + Reprovados + Abandono) × 100  
    - *Taxa de Reprovação (%)* = Reprovados / (Aprovados + Reprovados + Abandono) × 100  
    - *Taxa de Abandono (%)* = Abandonos / (Aprovados + Reprovados + Abandono) × 100  

    **Escopo:** Rede **estadual** do ES, unidade de análise: **município**.  
    **Fontes:** INEP (microdados e taxas agregadas).  
    **Próximos passos:** drill-down por escola, mapas, IDEB/SAEB, infraestrutura.
    """)
