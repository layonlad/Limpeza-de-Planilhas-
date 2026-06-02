# Limpeza-de-Planilhas-

"""Mapeamento de cargos para a lista oficial da Sólides (Etapa 7 da skill).

Lógica em 4 camadas:
  1. Correspondência exata normalizada (contra a lista oficial)
  2. Dicionário MAPA (sinônimos conhecidos)
  3. Busca por palavras-chave (nível + departamento RH/DP)
  4. Fallback -> "Outros"
"""

import re
from .texto import normalizar

# --------------------------------------------------------------------------
# Lista oficial — 23 valores permitidos
# --------------------------------------------------------------------------
CARGOS_OFICIAIS = [
    "Buscando recolocação",
    "Analista de RH",
    "Assistente de RH",
    "Business Partner de RH",
    "Coordenador de RH",
    "Diretor de RH",
    "Gerente de RH",
    "Sócio/CEO",
    "Coach",
    "Supervisor de RH",
    "Consultor de RH",
    "Estagiário de RH",
    "Contador",
    "Estudante",
    "Professor",
    "Outros",
    "Analista DP",
    "Administrativo/Financeiro",
    "Coordenador de DP",
    "Diretor de DP",
    "Gerente de DP",
    "Estagiário de DP",
    "Consultor de DP",
]

# --------------------------------------------------------------------------
# Dicionário MAPA — sinônimos conhecidos (chaves já normalizadas)
# --------------------------------------------------------------------------
MAPA = {
    # Analista RH
    "analista de rh": "Analista de RH",
    "analista rh": "Analista de RH",
    "analista de recursos humanos": "Analista de RH",
    "especialista de rh": "Analista de RH",
    "analista de sgi": "Analista de RH",
    "analista rhbp": "Analista de RH",
    # Analista DP
    "analista dp": "Analista DP",
    "analista de departamento pessoal": "Analista DP",
    "analista de pessoal": "Analista DP",
    # Assistente RH
    "assistente de rh": "Assistente de RH",
    "auxiliar de rh": "Assistente de RH",
    "auxiliar administrativo de rh": "Assistente de RH",
    "assistente de pessoal": "Assistente de RH",
    "auxiliar de departamento pessoal": "Assistente de RH",
    "assistente de departamento pessoal": "Assistente de RH",
    "assistente de saude ocupacional": "Assistente de RH",
    # Coordenador
    "coordenador de rh": "Coordenador de RH",
    "coordenadora de rh": "Coordenador de RH",
    "coord dp": "Coordenador de DP",
    "coordenador de dp": "Coordenador de DP",
    # Diretor
    "diretor de rh": "Diretor de RH",
    "head de rh": "Diretor de RH",
    "vp de rh": "Diretor de RH",
    # Gerente
    "gerente de rh": "Gerente de RH",
    "gerente administrativa e financeira": "Gerente de RH",
    # Supervisor
    "supervisor de rh": "Supervisor de RH",
    "supervisora de rh": "Supervisor de RH",
    # Business Partner
    "business partner de rh": "Business Partner de RH",
    "hrbp": "Business Partner de RH",
    "hr business partner": "Business Partner de RH",
    "especialista rhbp": "Business Partner de RH",
    # Consultor
    "consultor de rh": "Consultor de RH",
    "consultora de rh": "Consultor de RH",
    # Estagiário
    "estagiario de rh": "Estagiário de RH",
    "estagiaria de rh": "Estagiário de RH",
    "estagiario de dp": "Estagiário de DP",
    # Sócio/CEO
    "ceo": "Sócio/CEO",
    "proprietario": "Sócio/CEO",
    "empresario": "Sócio/CEO",
    # Outros diretos
    "secretaria": "Administrativo/Financeiro",
    "secretaria executiva": "Administrativo/Financeiro",
    "recepcionista": "Administrativo/Financeiro",
    "buscando recolocacao": "Buscando recolocação",
    "em busca de recolocacao": "Buscando recolocação",
}

# --------------------------------------------------------------------------
# Camada 3 — palavras-chave
# --------------------------------------------------------------------------
# Cada tupla: (lista_de_termos, resultado_RH, resultado_DP)
# resultado_DP = None quando não existe variante de DP na lista oficial.
_NIVEIS = [
    (["analista", "especialista"], "Analista de RH", "Analista DP"),
    (["assistente", "auxiliar"], "Assistente de RH", "Assistente de RH"),
    (["coordenador", "coordenadora", "coord"], "Coordenador de RH", "Coordenador de DP"),
    (["diretor", "diretora", "head", "vp"], "Diretor de RH", "Diretor de DP"),
    (["gerente"], "Gerente de RH", "Gerente de DP"),
    (["supervisor", "supervisora"], "Supervisor de RH", "Supervisor de RH"),
    (["consultor", "consultora"], "Consultor de RH", "Consultor de DP"),
    (["estagiario", "estagiaria", "trainee"], "Estagiário de RH", "Estagiário de DP"),
]

# Termos que mapeiam direto, independentemente de departamento.
_DIRETOS = [
    (["ceo", "socio", "proprietario", "empresario", "fundador",
      "diretor executivo", "presidente", "dono"], "Sócio/CEO"),
    (["recolocacao"], "Buscando recolocação"),
    (["contador", "contabil", "contabilidade"], "Contador"),
    (["professor", "docente"], "Professor"),
    (["estudante", "universitario", "graduando"], "Estudante"),
    (["coach"], "Coach"),
    (["business partner", "hrbp"], "Business Partner de RH"),
    (["recepcionista", "secretaria", "administrativo", "financeiro"],
     "Administrativo/Financeiro"),
]


def _tem_dp(norm):
    return bool(re.search(r"\bdp\b", norm)) or "departamento pessoal" in norm


def _tem_rh(norm):
    return bool(re.search(r"\brh\b", norm)) or "recursos humanos" in norm


def _busca_palavras_chave(norm):
    # Termos diretos primeiro.
    for termos, resultado in _DIRETOS:
        if any(t in norm for t in termos):
            return resultado

    # Níveis só valem se houver contexto de RH ou DP.
    dp = _tem_dp(norm)
    rh = _tem_rh(norm)
    if not (dp or rh):
        return None

    for termos, res_rh, res_dp in _NIVEIS:
        if any(t in norm for t in termos):
            if dp and res_dp:
                return res_dp
            return res_rh
    return None


def mapear_cargo(texto):
    """Mapeia um cargo livre para um dos 23 valores oficiais da Sólides."""
    norm = normalizar(texto)
    if not norm:
        return "Outros"

    # 1. Correspondência exata contra a lista oficial.
    for cargo in CARGOS_OFICIAIS:
        if normalizar(cargo) == norm:
            return cargo

    # 2. Dicionário MAPA.
    if norm in MAPA:
        return MAPA[norm]

    # 3. Palavras-chave.
    resultado = _busca_palavras_chave(norm)
    if resultado:
        return resultado

    # 4. Fallback.
    return "Outros"
