# Importações necessárias
import mysql.connector  # Para conexão com MySQL (instale com pip install mysql-connector-python)
from fpdf import FPDF   # Para gerar PDF (instale com pip install fpdf)
import pywhatkit        # Para envio via WhatsApp (instale com pip install pywhatkit)
import os
import datetime

# Linha de comando para anexar/vincular ao banco de dados (exemplo para teste via terminal)
# Rode isso no terminal para verificar conexão: mysql -h HOST -u USER -pPASSWORD DATABASE
# Em Python, a conexão é feita abaixo. Para automação, você pode usar subprocess para chamar comandos externos, mas aqui é via código.

# Detalhes de conexão com o banco de dados (EDITAR AQUI com seus dados reais)
DB_HOST = ''      # EDITAR AQUI: ex: 'localhost' ou 'seu-host.com'
DB_USER = ''      # EDITAR AQUI: ex: 'root' ou seu usuário
DB_PASSWORD = ''  # EDITAR AQUI: ex: 'sua-senha'
DB_NAME = ''      # EDITAR AQUI: ex: 'nome_do_banco'

# Placeholder para CPF (deixe em branco ou edite; será usado para busca no BD)
CPF = ''  # EDITAR AQUI: ex: '12345678901' (deixe vazio para input dinâmico)

# Placeholder para Número de WhatsApp (fallback se não encontrado no BD; formato +55XXYYYYYYYY)
NUMERO_WHATSAPP = ''  # EDITAR AQUI: ex: '+5511999999999' (deixe vazio para usar do BD)

def conectar_bd():
    try:
        conn = mysql.connector.connect(
            host=DB_HOST,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME
        )
        return conn
    except mysql.connector.Error as err:
        print(f"Erro ao conectar ao BD: {err}")
        return None

def buscar_dados_boleto(cpf):
    conn = conectar_bd()
    if not conn:
        return None, None
    
    cursor = conn.cursor()
    
    # Query assumindo tabelas 'boletos' e 'clientes' (ajuste se necessário)
    query = """
    SELECT b.data_geracao, b.valor, b.descricao, c.whatsapp
    FROM boletos b
    JOIN clientes c ON b.cpf = c.cpf
    WHERE b.cpf = %s
    LIMIT 1
    """
    
    cursor.execute(query, (cpf,))
    resultado = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    if resultado:
        return {
            'data_geracao': resultado[0],
            'valor': resultado[1],
            'descricao': resultado[2]
        }, resultado[3]  # Retorna dados do boleto e whatsapp
    else:
        print(f"Nenhum boleto encontrado para CPF: {cpf}")
        return None, None

def gerar_pdf_boleto(dados, cpf):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    
    pdf.cell(200, 10, txt="Boleto Gerado", ln=True, align='C')
    pdf.cell(200, 10, txt=f"CPF: {cpf}", ln=True)
    pdf.cell(200, 10, txt=f"Data de Geração: {dados['data_geracao']}", ln=True)
    pdf.cell(200, 10, txt=f"Valor: R$ {dados['valor']}", ln=True)
    pdf.cell(200, 10, txt=f"Descrição: {dados['descricao']}", ln=True)
    
    nome_arquivo = f"boleto_{cpf}.pdf"
    pdf.output(nome_arquivo)
    print(f"PDF gerado: {nome_arquivo}")
    return nome_arquivo

def enviar_whatsapp(numero, arquivo_pdf):
    if not numero:
        print("Número de WhatsApp não encontrado ou não editado.")
        return
    
    # Envia mensagem com anexo (pywhatkit envia texto; para anexo, usamos automação simples)
    # Nota: Abra WhatsApp Web no navegador antes de rodar.
    hora_atual = datetime.datetime.now().hour
    minuto_atual = datetime.datetime.now().minute + 1  # Envia em 1 minuto
    
    pywhatkit.sendwhatmsg(numero, "Aqui está o seu boleto em PDF!", hora_atual, minuto_atual)
    # Para anexo, pywhatkit não suporta diretamente; use sendwhats_image para imagem ou adapte com selenium.
    # Alternativa simples: printa instrução (para produção, integre com Selenium).
    print(f"Mensagem enviada para {numero}. Para anexar PDF manualmente, use WhatsApp Web.")
    # TODO: Se precisar de anexo automático, instale selenium e adicione código para automação.

if __name__ == "__main__":
    # Se CPF não editado, peça input
    if not CPF:
        CPF = input("Digite o CPF para busca: ").strip()
    
    dados_boleto, whatsapp_bd = buscar_dados_boleto(CPF)
    
    if dados_boleto:
        arquivo_pdf = gerar_pdf_boleto(dados_boleto, CPF)
        
        # Usa número do BD ou fallback editável
        numero_enviar = whatsapp_bd if whatsapp_bd else NUMERO_WHATSAPP
        enviar_whatsapp(numero_enviar, arquivo_pdf)
        
        # Limpa o PDF após envio (opcional)
        # os.remove(arquivo_pdf)
    else:
        print("Operação cancelada: boleto não encontrado.")
