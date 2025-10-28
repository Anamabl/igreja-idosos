# Arquivo: app.py (Sistema Conecta Fé)

from flask import Flask, render_template, request, redirect, url_for
import json
import os
import datetime

# --- Configurações Iniciais ---
app = Flask(__name__)
DATA_FILE = "dados_comunidade.json"

# --- Funções de Persistência ---

def carregar_dados():
    """Carrega dados da comunidade (eventos, suporte, voluntários) do JSON."""
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, 'r', encoding='utf-8') as f:
            try:
                return json.load(f)
            except json.JSONDecodeError:
                # Retorna estrutura inicial em caso de arquivo corrompido
                print("🚨 Arquivo de dados corrompido. Reiniciando dados.")
    
    # Estrutura inicial do banco de dados (JSON) com IDs sequenciais
    return {
        'proximo_id_suporte': 1,
        'proximo_id_evento': 1,
        'eventos': [],
        'suportes': [],
        'voluntarios': []
    }

def salvar_dados(dados):
    """Salva a estrutura completa de dados no arquivo JSON."""
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(dados, f, indent=4, ensure_ascii=False)

# --- Rotas da Aplicação ---

@app.route('/')
def index():
    """Página inicial: Painel de Avisos/Agenda da Igreja. Ordena por data."""
    dados = carregar_dados()
    
    # Lógica de ordenação robusta por data
    try:
        # Tenta ordenar corretamente por data (data mais antiga primeiro)
        eventos_ordenados = sorted(
            dados.get('eventos', []),
            key=lambda e: datetime.date.fromisoformat(e.get('data', '2999-01-01')) # Usa uma data futura para itens sem data
        )
    except Exception:
        # Fallback: ordena por string se houver formato inesperado
        print("Aviso: Houve erro na ordenação por data. Usando ordenação por string.")
        eventos_ordenados = sorted(dados.get('eventos', []), key=lambda e: e.get('data', ''), reverse=True)
    
    return render_template('index.html', eventos=eventos_ordenados)

@app.route('/solicitar-suporte', methods=['GET', 'POST'])
def solicitar_suporte():
    """Página para membros solicitarem ajuda técnica (IDosos/Família)."""
    dados = carregar_dados()
    
    if request.method == 'POST':
        # Coleta os dados do formulário de solicitação de ajuda
        nome_membro = request.form['nome']
        contato = request.form['contato']
        descricao = request.form['descricao']
        
        novo_suporte = {
            'id': dados['proximo_id_suporte'],
            'data': datetime.date.today().isoformat(),
            'nome_membro': nome_membro,
            'contato': contato,
            'descricao': descricao,
            'status': 'Pendente',
            'voluntario_id': None
        }
        
        dados['suportes'].append(novo_suporte)
        dados['proximo_id_suporte'] += 1
        salvar_dados(dados)
        
        print(f"✅ Nova solicitação de suporte ID {novo_suporte['id']} registrada.")
        return redirect(url_for('confirmacao_suporte'))
        
    return render_template('solicitar_suporte.html')

@app.route('/confirmacao-suporte')
def confirmacao_suporte():
    """Página de confirmação após o envio da solicitação."""
    return render_template('confirmacao_suporte.html')

# --- Rotas de Administração (Adicionar Eventos) ---

@app.route('/admin/eventos/novo', methods=['GET', 'POST'])
def adicionar_evento():
    """Rota para a liderança adicionar novos eventos à agenda, com gestão de ID."""
    dados = carregar_dados()
    
    if request.method == 'POST':
        # Adiciona o evento com ID incremental e incrementa o contador
        novo_evento = {
            'id': dados.get('proximo_id_evento', 1),
            'titulo': request.form['titulo'],
            'data': request.form['data'],
            'link_transmissao': request.form.get('link_transmissao', '')
        }
        dados['eventos'].append(novo_evento)
        # Incrementa o próximo ID de evento para o próximo cadastro
        dados['proximo_id_evento'] = dados.get('proximo_id_evento', 1) + 1 
        salvar_dados(dados)
        return redirect(url_for('index'))
    
    # Rota GET para exibir o formulário (retorno HTML puro para simplicidade)
    return """
        <h2>Adicionar Novo Evento</h2>
        <form method="POST">
            Título: <input type="text" name="titulo" required><br>
            Data (AAAA-MM-DD): <input type="date" name="data" required><br>
            Link (Zoom/YouTube): <input type="url" name="link_transmissao"><br>
            <input type="submit" value="Salvar Evento">
        </form>
        <p><a href="/">Voltar para a Agenda</a></p>
    """

# --- Execução da Aplicação e Criação de Templates (Setup) ---

def setup_files():
    """Garante que a pasta 'templates' e os arquivos iniciais existam."""
    
    # Cria pasta 'templates' se não existir
    if not os.path.exists('templates'):
        os.makedirs('templates')
    
    # 1. index.html (Painel Principal)
    with open('templates/index.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Conecta Fé - Agenda</title></head>
        <body>
            <h1>Conecta Fé: Agenda Comunitária</h1>
            <p>Seja bem-vindo(a)! Abaixo, a agenda de eventos da nossa comunidade.</p>
            <p><a href="/solicitar-suporte">👵🏼 Precisa de Ajuda Técnica? Clique aqui!</a></p>
            <p><a href="/admin/eventos/novo">Adicionar Novo Evento (Admin)</a></p>
            
            <h2>Próximos Eventos</h2>
            {% for evento in eventos %}
                <div style="border: 1px solid #ccc; margin-bottom: 15px; padding: 15px; border-radius: 5px;">
                    <h3>{{ evento.titulo }} (ID: {{ evento.id }})</h3>
                    <p><strong>Data:</strong> {{ evento.data }}</p>
                    {% if evento.link_transmissao %}
                        <p><strong>Link:</strong> <a href="{{ evento.link_transmissao }}">Acessar Transmissão Online</a></p>
                    {% endif %}
                </div>
            {% else %}
                <p>Nenhum evento agendado no momento.</p>
            {% endfor %}
        </body>
        </html>
        """)

    # 2. solicitar_suporte.html (Formulário de Pedido de Ajuda)
    with open('templates/solicitar_suporte.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Solicitar Suporte</title></head>
        <body>
            <h2>Solicitar Suporte Técnico Gratuito</h2>
            <p>Preencha os dados abaixo. Um voluntário entrará em contato para ajudar com:</p>
            <ul><li>Configuração de celular/tablet.</li><li>Acesso a lives e links.</li><li>Outras dificuldades digitais.</li></ul>
            <form method="POST">
                Seu Nome: <input type="text" name="nome" required><br><br>
                Contato (Telefone/Email): <input type="text" name="contato" required><br><br>
                Descreva a dificuldade (Ex: Não consigo abrir o link da live): <textarea name="descricao" required></textarea><br><br>
                <input type="submit" value="Enviar Pedido de Ajuda">
            </form>
            <p><a href="/">Voltar para a Agenda</a></p>
        </body>
        </html>
        """)

    # 3. confirmacao_suporte.html (Confirmação)
    with open('templates/confirmacao_suporte.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Confirmação</title></head>
        <body>
            <h2>✅ Pedido Enviado!</h2>
            <p>Sua solicitação de suporte foi registrada com sucesso. Agradecemos sua paciência. Em breve, um voluntário da comunidade entrará em contato.</p>
            <p><a href="/">Voltar para a Agenda</a></p>
        </body>
        </html>
        """)


def inicializar_dados_exemplo(dados):
    """Adiciona um evento de exemplo na primeira execução se a lista de eventos estiver vazia."""
    if not dados.get('eventos'):
        dados['eventos'].append({
            'titulo': 'Culto Dominical - Online',
            'data': datetime.date.today().isoformat(),
            'link_transmissao': 'http://youtube.com/live',
            'id': dados.get('proximo_id_evento', 1)
        })
        dados['proximo_id_evento'] = dados.get('proximo_id_evento', 1) + 1
        salvar_dados(dados)

if __name__ == '__main__':
    # 1. Garante que os arquivos HTML estejam prontos
    setup_files()
    
    # 2. Carrega dados e inicializa exemplos
    dados = carregar_dados()
    inicializar_dados_exemplo(dados)
    
    # 3. Inicia a aplicação Flask
    print("\n--- Conecta Fé Iniciado ---")
    print("Acesse: http://127.0.0.1:5000/")
    print("Para adicionar eventos (Admin): http://127.0.0.1:5000/admin/eventos/novo")
    print("---------------------------\n")
    
    app.run(debug=True)
