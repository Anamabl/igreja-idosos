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
    
    # Estrutura inicial do banco de dados (JSON) - COM proximo_id_evento
    return {
        'proximo_id_suporte': 1,
        'proximo_id_evento': 1,  # Novo ID para eventos
        'eventos': [],
        'suportes': [],
        'voluntarios': []
    }

def salvar_dados(dados):
    """Salva a estrutura completa de dados no JSON."""
    with open(DATA_FILE, 'w', encoding='utf-8') as f:
        json.dump(dados, f, indent=4, ensure_ascii=False)

# --- Rotas da Aplicação ---

@app.route('/')
def index():
    """Página inicial: Painel de Avisos/Agenda da Igreja."""
    dados = carregar_dados()
    
    # Ordena eventos por data (converte ISO para objeto date)
    try:
        # Tenta ordenar corretamente por data (data mais antiga primeiro)
        eventos_ordenados = sorted(
            dados.get('eventos', []),
            key=lambda e: datetime.date.fromisoformat(e.get('data', '2999-01-01'))
        )
    except Exception:
        # Fallback: ordena por string se houver formato inesperado (para robustez)
        print("Aviso: Houve erro na ordenação por data. Usando ordenação por string.")
        eventos_ordenados = sorted(dados.get('eventos', []), key=lambda e: e.get('data', ''), reverse=True)
    
    return render_template('index.html', eventos=eventos_ordenados)

@app.route('/solicitar-suporte', methods=['GET', 'POST'])
def solicitar_suporte():
    """Página para membros solicitarem ajuda técnica."""
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

# --- Rotas de Administração (Exemplo de CRUD) ---

@app.route('/admin/eventos/novo', methods=['GET', 'POST'])
def adicionar_evento():
    """Rota para a liderança adicionar novos eventos à agenda."""
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
        # Incrementa o próximo ID de evento
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

# --- Execução da Aplicação ---
if __name__ == '__main__':
    # Garante que os dados iniciais sejam carregados ou criados
    dados = carregar_dados()
    
    # Exemplo de inicialização de dados na primeira execução (simulação de um evento)
    if not dados.get('eventos'):
        dados['eventos'].append({
            'titulo': 'Culto Dominical - Online',
            'data': datetime.date.today().isoformat(),
            'link_transmissao': 'http://youtube.com/live',
            'id': dados.get('proximo_id_evento', 1)
        })
        dados['proximo_id_evento'] = dados.get('proximo_id_evento', 1) + 1
        salvar_dados(dados)

    # Nota: Em um ambiente real, você criaria os templates HTML abaixo.
    # Certifique-se de ter uma pasta 'templates' com estes arquivos.
    
    # 1. Criação de templates simples para que o código funcione
    if not os.path.exists('templates'):
        os.makedirs('templates')
    
    # index.html
    with open('templates/index.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Conecta Fé - Agenda</title></head>
        <body>
            <h1>Conecta Fé: Agenda Comunitária</h1>
            <p><a href="/solicitar-suporte">Precisa de Ajuda Técnica? Clique aqui!</a></p>
            <p><a href="/admin/eventos/novo">Adicionar Novo Evento (Admin)</a></p>
            
            <h2>Próximos Eventos</h2>
            {% for evento in eventos %}
                <div style="border: 1px solid #ccc; margin-bottom: 10px; padding: 10px;">
                    <h3>{{ evento.titulo }} (ID: {{ evento.id }})</h3>
                    <p><strong>Data:</strong> {{ evento.data }}</p>
                    {% if evento.link_transmissao %}
                        <p><strong>Link:</strong> <a href="{{ evento.link_transmissao }}">Acessar Transmissão</a></p>
                    {% endif %}
                </div>
            {% else %}
                <p>Nenhum evento agendado.</p>
            {% endfor %}
        </body>
        </html>
        """)

    # solicitar_suporte.html
    with open('templates/solicitar_suporte.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Solicitar Suporte</title></head>
        <body>
            <h2>Solicitar Suporte Técnico</h2>
            <form method="POST">
                Seu Nome: <input type="text" name="nome" required><br>
                Contato (Telefone/Email): <input type="text" name="contato" required><br>
                Descreva a dificuldade (Ex: Não consigo abrir o link da live): <textarea name="descricao" required></textarea><br>
                <input type="submit" value="Enviar Pedido">
            </form>
            <p><a href="/">Voltar para a Agenda</a></p>
        </body>
        </html>
        """)

    # confirmacao_suporte.html
    with open('templates/confirmacao_suporte.html', 'w', encoding='utf-8') as f:
        f.write("""
        <!doctype html>
        <html lang="pt-br">
        <head><title>Confirmação</title></head>
        <body>
            <h2>✅ Pedido Enviado!</h2>
            <p>Sua solicitação de suporte foi registrada. Um voluntário será notificado e entrará em contato em breve.</p>
            <p><a href="/">Voltar para a Agenda</a></p>
        </body>
        </html>
        """)

    # Inicia a aplicação
    app.run(debug=True)
