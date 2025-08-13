
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8" />
<title>Backup na Nuvem Local</title>
<style>
  body {
    font-family: Arial, sans-serif;
    margin: 40px;
    background: #e0f7ff;
    text-align: center;
  }
  #cloud {
    width: 150px;
    margin: 0 auto;
    cursor: pointer;
    position: relative;
    animation: float 3s ease-in-out infinite;
  }
  @keyframes float {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-15px); }
  }
  #fileInput {
    margin: 20px 0;
  }
  button {
    padding: 10px 25px;
    background: #007bff;
    border: none;
    color: white;
    border-radius: 25px;
    cursor: pointer;
    font-size: 16px;
    transition: background 0.3s ease;
    margin: 10px 5px;
  }
  button:hover {
    background: #0056b3;
  }
  #status {
    margin-top: 20px;
    color: green;
    font-weight: bold;
  }
  #fileList {
    max-width: 400px;
    margin: 20px auto 0;
    background: white;
    border-radius: 10px;
    box-shadow: 0 1px 6px rgba(0,0,0,0.15);
    padding: 15px;
    text-align: left;
    display: none;
  }
  #fileList h2 {
    text-align: center;
    margin-top: 0;
  }
  #fileList ul {
    list-style: none;
    padding-left: 0;
  }
  #fileList li {
    margin: 8px 0;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  #fileList button.download-btn {
    padding: 4px 8px;
    font-size: 14px;
    border-radius: 5px;
    background: #28a745;
  }
  #fileList button.delete-btn {
    background: #dc3545;
    margin-left: 10px;
  }
  #downloadByNameSection {
    margin-top: 30px;
  }
  #downloadByNameSection input {
    padding: 8px;
    font-size: 16px;
    width: 220px;
    border-radius: 5px;
    border: 1px solid #ccc;
  }
</style>
</head>
<body>

<!-- Nuvem SVG clicável -->
<svg id="cloud" viewBox="0 0 64 39" xmlns="http://www.w3.org/2000/svg" aria-label="Ícone de nuvem">
  <g fill="#90caf9">
    <ellipse cx="24" cy="20" rx="14" ry="13" />
    <ellipse cx="42" cy="20" rx="14" ry="13" />
    <ellipse cx="33" cy="14" rx="18" ry="14" />
  </g>
  <ellipse cx="33" cy="26" rx="26" ry="12" fill="#bbdefb" />
</svg>

<h1>Backup na Nuvem Local</h1>
<p>Selecione um arquivo para fazer backup dentro da nuvem do site.</p>

<input type="file" id="fileInput" />
<br />
<button id="backupBtn">Fazer Backup</button>

<div id="status"></div>

<!-- Lista de arquivos dentro da nuvem -->
<div id="fileList">
  <h2>Arquivos na Nuvem</h2>
  <ul></ul>
</div>

<!-- Nova seção para baixar por nome -->
<div id="downloadByNameSection">
  <h3>Baixar arquivo pelo nome</h3>
  <input type="text" id="downloadFileName" placeholder="Digite o nome exato do arquivo" />
  <button id="downloadByNameBtn">Baixar</button>
</div>

<script>
  let db;
  const request = indexedDB.open('backupNuvemDB', 1);

  request.onerror = function(event) {
    alert('Erro ao abrir banco de dados IndexedDB');
  };

  request.onupgradeneeded = function(event) {
    db = event.target.result;
    const objectStore = db.createObjectStore('arquivos', { keyPath: 'id', autoIncrement: true });
    objectStore.createIndex('filename', 'filename', { unique: false });
  };

  request.onsuccess = function(event) {
    db = event.target.result;
    carregarArquivos();
  };

  const fileListDiv = document.getElementById('fileList');
  const fileListUl = fileListDiv.querySelector('ul');
  const statusDiv = document.getElementById('status');
  const cloudSVG = document.getElementById('cloud');

  document.getElementById('backupBtn').addEventListener('click', () => {
    const fileInput = document.getElementById('fileInput');
    if (fileInput.files.length === 0) {
      alert('Selecione um arquivo primeiro!');
      return;
    }
    const file = fileInput.files[0];
    const reader = new FileReader();
    reader.onload = function(event) {
      const arquivoData = {
        filename: file.name,
        type: file.type,
        data: event.target.result
      };
      salvarArquivo(arquivoData);
    };
    reader.readAsDataURL(file);
  });

  function salvarArquivo(arquivo) {
    const transaction = db.transaction(['arquivos'], 'readwrite');
    const objectStore = transaction.objectStore('arquivos');
    const requestAdd = objectStore.add(arquivo);

    requestAdd.onsuccess = function() {
      statusDiv.textContent = `Backup do arquivo "${arquivo.filename}" feito com sucesso!`;
      carregarArquivos();
      document.getElementById('fileInput').value = '';
    };

    requestAdd.onerror = function() {
      alert('Erro ao salvar arquivo');
    };
  }

  function carregarArquivos() {
    const transaction = db.transaction(['arquivos'], 'readonly');
    const objectStore = transaction.objectStore('arquivos');
    const requestGetAll = objectStore.getAll();

    requestGetAll.onsuccess = function() {
      const arquivos = requestGetAll.result;
      fileListUl.innerHTML = '';
      if (arquivos.length === 0) {
        fileListUl.innerHTML = '<li><i>Nenhum arquivo salvo ainda</i></li>';
      } else {
        arquivos.forEach(file => {
          const li = document.createElement('li');
          li.textContent = file.filename;

          const btnDownload = document.createElement('button');
          btnDownload.textContent = 'Download';
          btnDownload.classList.add('download-btn');
          btnDownload.onclick = () => baixarArquivo(file);

          const btnDelete = document.createElement('button');
          btnDelete.textContent = 'Excluir';
          btnDelete.classList.add('delete-btn');
          btnDelete.onclick = () => deletarArquivo(file.id);

          const btnDiv = document.createElement('div');
          btnDiv.appendChild(btnDownload);
          btnDiv.appendChild(btnDelete);

          li.appendChild(btnDiv);
          fileListUl.appendChild(li);
        });
      }
    };
  }

  function baixarArquivo(file) {
    const a = document.createElement('a');
    a.href = file.data;
    a.download = file.filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  function deletarArquivo(id) {
    const transaction = db.transaction(['arquivos'], 'readwrite');
    const objectStore = transaction.objectStore('arquivos');
    const requestDelete = objectStore.delete(id);

    requestDelete.onsuccess = function() {
      statusDiv.textContent = 'Arquivo excluído com sucesso!';
      carregarArquivos();
    };
  }

  // Mostrar / esconder lista arquivos clicando na nuvem
  cloudSVG.addEventListener('click', () => {
    if (fileListDiv.style.display === 'none' || fileListDiv.style.display === '') {
      fileListDiv.style.display = 'block';
    } else {
      fileListDiv.style.display = 'none';
    }
  });

  // Novo: baixar arquivo pelo nome exato
  document.getElementById('downloadByNameBtn').addEventListener('click', () => {
    const nome = document.getElementById('downloadFileName').value.trim();
    if (!nome) {
      alert('Digite o nome do arquivo para baixar');
      return;
    }
    const transaction = db.transaction(['arquivos'], 'readonly');
    const objectStore = transaction.objectStore('arquivos');
    const index = objectStore.index('filename');
    const arquivos = [];

    const request = index.openCursor();
    let found = false;

    request.onsuccess = function(event) {
      const cursor = event.target.result;
      if (cursor) {
        if (cursor.value.filename === nome) {
          found = true;
          baixarArquivo(cursor.value);
          document.getElementById('downloadFileName').value = '';
          statusDiv.textContent = `Arquivo "${nome}" baixado com sucesso!`;
          return;
        }
        cursor.continue();
      } else {
        if (!found) alert('Arquivo não encontrado com esse nome.');
      }
    };
  });
</script>

</body>
</html>
