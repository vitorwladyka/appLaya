import json
import requests

# 1. Define o endpoint apontando para o seu container Docker
URL_CONTAINER = "http://localhost:8001/v1/systemone"

# 2. Define o "state" (o payload de entrada que o modelo vai analisar)
ticket_cliente = {
    "mensagem": "Comprei o produto ontem, mas fui cobrado duas vezes no meu cartão. Quero o estorno."
}

# 3. Define as perguntas estruturadas (schema)
esquema_decisao = {
    "roteamento_departamento": {
        "type": "choice",
        "instructions": "Qual departamento deve tratar este chamado?",
        "criteria": {
            "financeiro": "Problemas com pagamentos, faturamento, duplicidade ou reembolsos.",
            "suporte_tecnico": "Erros no sistema, bugs, problemas de login ou falhas no produto.",
            "vendas": "Dúvidas sobre novos planos, preços ou interesse em adquirir o produto."
        }
    }
}

# 4. Monta o corpo da requisição esperado pelo laya-serve
payload = {
    "state": ticket_cliente,
    "questions": esquema_decisao
}

headers = {
    "Content-Type": "application/json"
}

try:
    # 5. Envia a requisição HTTP POST para o container
    resposta = requests.post(URL_CONTAINER, headers=headers, json=payload)
    resposta.raise_for_status()  # Levanta erro caso o container retorne HTTP 4xx ou 5xx
    
    # 6. Processa a resposta estruturada vinda do container
    dados_resposta = resposta.json()
    
    # DESCOMENTE A LINHA ABAIXO CASO QUEIRA VER A ESTRUTURA EXATA QUE O CONTAINER RETORNOU:
    # print("RESPOSTA REAL DO CONTAINER:\n", json.dumps(dados_resposta, indent=2))
    
    # Captura a raiz das respostas (algumas versões envelopam em 'answers', outras trazem na raiz)
    decisoes = dados_resposta.get("answers", dados_resposta)
    decisao_final = decesoes_mapeadas = decisoes.get("roteamento_departamento")
    
    if decisao_final is None:
        print("Erro: A chave 'roteamento_departamento' não foi encontrada na resposta do container.")
    else:
        # Tratamento seguro para diferentes versões do laya-serve (dicionário vs string direta)
        if isinstance(decisao_final, dict):
            opcao_selecionada = decisao_final.get("selection") or decisao_final.get("value")
            confianca = decisao_final.get("confidence", 0.0)
            probabilidades = decisao_final.get("probabilities", {})
        else:
            # Se a resposta vier simplificada diretamente como string
            opcao_selecionada = decisao_final
            confianca = 1.0
            probabilidades = {}

        # 7. Exibe os resultados tratados
        print(f"Opção selecionada: {opcao_selecionada}")
        print(f"Grau de confiança: {confianca:.4f}")
        
        if probabilidades:
            print("\nDistribuição das probabilidades:")
            for opcao, probabilidade in probabilidades.items():
                print(f" - {opcao}: {probabilidade:.4f}")

except requests.exceptions.RequestException as e:
    print(f"Erro ao conectar com o container do Laya: {e}")
