# RF-LogFilter: filtragem de logs de segurança com Random Forest para o AgCyRAG

| | |
|---|---|
| **Disciplina** | Aprendizagem de Máquina |
| **Instituição** | Faculdade de Tecnologia de Mauá (Fatec Mauá) |
| **Professor** | Professor Dr. Bruno Zolotareff dos Santos |
| **Grupo** | Anderson Mendes Leandro, Ismael Alves de Almeida, Rafael Massayoshi Targa Eda |

## Objetivo e problema

O artigo base propõe o **AgCyRAG**, um framework de RAG com agentes e grafos de conhecimento para análise de segurança. Nele, logs brutos (AIT Log Data Set) são convertidos em grafo no Neo4j por um construtor baseado em LLM, e agentes consultam esse grafo para responder perguntas de um analista.

**Problema.** Todo log entra na base de conhecimento, inclusive o ruído operacional (CRON, sessões rotineiras, logins normais). Como a construção do grafo usa um LLM, o custo cresce com o volume, e o ruído concorre com a evidência relevante na hora da recuperação. O próprio artigo aponta escalabilidade como trabalho futuro.

**Objetivo.** Adicionar uma etapa de aprendizado de máquina antes da ingestão: um classificador **Random Forest**, implementado do zero, que dá a cada linha de log um *score de risco*. Só as linhas mais parecidas com um incidente real (e o contexto próximo delas) seguem para a base de conhecimento.

## Solução prévia

```
logs brutos -> parser -> features de contexto -> Random Forest -> score de risco -> filtro -> grafo de logs (AgCyRAG)
```

1. **Parser** (regex): extrai data/hora, processo, tipo de evento, usuário, IP e comando de cada linha do `auth.log`.
2. **Features**: 22 atributos por linha, combinando o evento em si (falha, sucesso, sudo, horário, IP externo) com contexto calculado só a partir do passado (falhas do mesmo IP em 1, 10 e 60 minutos, usuários distintos por IP, IPs distintos por usuário, etc.).
3. **Random Forest do zero**: árvores CART com impureza de Gini, amostragem *bootstrap*, sorteio de √n features por nó, média das probabilidades, validação *out-of-bag* e importância de features. Sem scikit-learn; apenas `numpy` e `matplotlib`.
4. **Filtro**: mantém as linhas com score acima de um limiar escolhido no treino e também as linhas do mesmo usuário/IP até 2 minutos antes ou depois. O score é gravado junto com a linha para virar uma propriedade (`risk_score`) no grafo.

## MVP (Colab)

Notebook: [`RF_LogFilter_AgCyRAG.ipynb`](RF_LogFilter_AgCyRAG.ipynb). Abrir no Google Colab e usar *Executar tudo*; não precisa instalar nada.

**Dados.** O notebook gera uma massa sintética rotulada (`auth.log` + `auth.log.labels`) no formato do caso de uso do artigo: 6 dias de um servidor `mail-0`, cerca de 15 mil linhas, 12% de ataque. Inclui força bruta, varredura, *credential stuffing*, *password spraying* lento, ataque distribuído, ataque interno e ações pós-invasão, além de casos benignos difíceis (usuário que erra a senha, conta de serviço com senha vencida, home office).

**Resultados no conjunto de teste** (treino nos 4 primeiros dias, teste nos 2 últimos):

| Abordagem | Precisão | Recall | Redução de volume |
|---|---|---|---|
| Sem filtro (artigo original) | 12,4% | 100% | 0% |
| Regra fixa: manter toda falha de login | 73,8% | 96,6% | 83,8% |
| Uma árvore de decisão | 98,6% | 93,7% | 88,2% |
| Random Forest (limiar 0,5) | 98,5% | 98,4% | 87,6% |
| Random Forest + contexto (filtro final) | — | 99,6% | 87,3% |

## Limitações

- Os dados são sintéticos e gerados pelo próprio grupo, então o modelo em parte reaprende as regras do gerador. Os números provam o pipeline, não a eficácia em ambiente real.
- Uma regra simples já chega perto; o ganho do Random Forest está na precisão e nos casos que a regra não cobre (ações pós-invasão).
- Filtrar é irreversível: o que for descartado fica invisível para os agentes. Recomendamos guardar o log bruto completo e usar o score como prioridade.
- Não medimos o efeito do filtro nas respostas do AgCyRAG.

## Próximos passos

- Validar com o AIT Log Data Set v2 (o mesmo do artigo), que tem rótulos por linha.
- Integrar `auth_filtrado.jsonl` à ingestão do AgCyRAG e comparar custo e qualidade das respostas com e sem filtro.

## Referências

1. KURNIAWAN, K.; ARDIAN, R. F.; KIESLING, E.; EKELHART, A. AgCyRAG: an Agentic Knowledge Graph based RAG Framework for Automated Security Analysis. In: RAGE-KG 2025, co-located with ISWC 2025, Nara, Japão, 2025.
2. BREIMAN, L. Random Forests. *Machine Learning*, v. 45, n. 1, p. 5–32, 2001.
3. LANDAUER, M.; SKOPIK, F.; FRANK, M.; HOTWAGNER, W.; WURZENBERGER, M.; RAUBER, A. Maintainable Log Datasets for Evaluation of Intrusion Detection Systems. *IEEE Transactions on Dependable and Secure Computing*, 2023. arXiv:2203.08580. Dados: AIT Log Data Set V2.0, Zenodo, https://zenodo.org/record/5789064.
