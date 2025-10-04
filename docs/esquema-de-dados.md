
#  Esquema de Dados — Circular Raw Material Calculator

> Estrutura técnica e formato dos dados utilizados pela Calculadora Circular de Matéria-Prima.  
> Define os objetos, campos, unidades e formatos de intercâmbio, assegurando compatibilidade com integrações futuras (API ou dashboards).

## 1) Objetos Principais

### 1.1 Company (Diagnóstico)
```json
{
  "companyId": "diag_01HZX...",
  "name": "Empresa Exemplo Lda",
  "taxId": "509999999",
  "sector": "Turismo",
  "period": "2024",
  "unitsProduced": 10000,
  "unitsSold": 9500,
  "unitWeightKg": 0.5
}
```

### 1.2 Material (Materiais)
```json
{
  "materialId": "mat_01HZ...",
  "companyId": "diag_01HZX...",
  "name": "PET",
  "use": "Embalagem",
  "inputQtyKg": 5000,
  "recycledInputPct": 10.0,
  "processLossPct": 5.0,
  "residueKg": 400,
  "destiny": "Reciclagem externa",
  "eolCollectionPct": 30.0,
  "costPerKg": 1.2,
  "disposalCostPerKg": 0.3,
  "energyKwhPerKg": 2.5,
  "risk1to5": 4
}
```

### 1.3 ESG Checklist (Checklist ESG)
```json
{
  "checkId": "esg_01...",
  "companyId": "diag_01HZX...",
  "pillar": "Ambiental",
  "question": "Política Ambiental Formal?",
  "answer": "Sim"
}
```

### 1.4 Results (Resultados & KPIs)
```json
{
  "resultId": "res_01...",
  "companyId": "diag_01HZX...",
  "circularityInputPct": 15.0,
  "valorizationPct": 40.0,
  "endOfLifeCollectionPct": 25.0,
  "wastePerUnitKg": 0.05,
  "materialCostPerUnitEur": 0.60,
  "lossCostPerUnitEur": 0.10,
  "hotspots": ["PET", "Papel", "Energia"],
  "scoreE": 55,
  "scoreS": 60,
  "scoreG": 50,
  "scoreGlobal": 55,
  "tier": "Em Desenvolvimento"
}
```
