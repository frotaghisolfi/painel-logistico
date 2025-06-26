<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Painel Logístico - Avançado</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <style>
    body {
      font-family: 'Segoe UI', sans-serif;
      background-color: #f4f6f8;
      padding: 30px;
    }
    h1 {
      text-align: center;
    }
    section {
      background: #fff;
      border-radius: 8px;
      padding: 20px;
      margin: 20px auto;
      max-width: 1100px;
      box-shadow: 0 0 10px rgba(0,0,0,0.05);
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 10px;
    }
    th, td {
      padding: 10px;
      border: 1px solid #ccc;
      text-align: center;
    }
    input, select, button {
      margin: 5px;
      padding: 8px;
      border-radius: 5px;
    }
    button {
      background-color: #4e73df;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #375ac2;
    }
    canvas {
      margin-top: 20px;
    }
  </style>
</head>
<body>

<h1>Painel Logístico Unificado - Dados & Desempenho</h1>

<section>
  <h2>📂 Importar Relatórios e Gerar Dados</h2>
  <input type="file" id="fileInput" accept=".csv" />
  <button onclick="gerarDadosExemplo()">Gerar Dados Simulados</button>
  <button onclick="exportarPDF()">Exportar para PDF 🧾</button>
</section>

<section>
  <h2>🔍 Filtros</h2>
  <label>Região:
    <select id="filtroRegiao" onchange="aplicarFiltros()">
      <option value="">Todas</option>
    </select>
  </label>
  <label>Situação:
    <select id="filtroSituacao" onchange="aplicarFiltros()">
      <option value="">Todas</option>
      <option value="Roteirizado">Roteirizado</option>
      <option value="Não Roteirizado">Não Roteirizado</option>
    </select>
  </label>
  <label>Data Inicial:
    <input type="date" id="filtroDataInicio" onchange="aplicarFiltros()" />
  </label>
  <label>Data Final:
    <input type="date" id="filtroDataFim" onchange="aplicarFiltros()" />
  </label>
</section>

<section>
  <h2>🚚 Veículos e Entregas</h2>
  <table id="tabelaVeiculos">
    <thead>
      <tr>
        <th>Data</th>
        <th>Nº do Mapa</th>
        <th>Hora Saída</th>
        <th>Região</th>
        <th>Situação</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
</section>

<section>
  <h2>💰 Painel Financeiro</h2>
  <table id="tabelaFinanceira">
    <thead>
      <tr>
        <th>Segmento</th>
        <th>Desconto (R$)</th>
        <th>Dias Disponíveis</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
  <canvas id="graficoDescontos" height="100"></canvas>
</section>

<script>
  let dadosVeiculos = [], dadosFinanceiros = [], dadosFiltrados = [];

  document.getElementById('fileInput').addEventListener('change', function (e) {
    const arquivo = e.target.files[0];
    if (!arquivo) return;
    Papa.parse(arquivo, {
      header: true,
      skipEmptyLines: true,
      complete: function (result) {
        const rows = result.data;
        if ('Nro do Mapa' in rows[0]) {
          dadosVeiculos = rows.map(r => ({
            Data: r['Data Entrega'] || '',
            Mapa: r['Nro do Mapa'] || '',
            "Hora Saída": r['Hora Imp Roadshow'] || '',
            Região: r['Região +Entregas'] || '',
            Situação: r['Roteirizado'] || 'Não Roteirizado'
          }));
          preencherFiltros();
          aplicarFiltros();
        } else if ('Frota Disp. Freteiro Disp.' in rows[0]) {
          dadosFinanceiros = rows.map(r => ({
            Segmento: r['Frota Disp. Freteiro Disp.'],
            "Desconto FA (R$)": parseFloat((r['Desconto FA'] || '0').replace('.', '').replace(',', '.')),
            "Dias Disponíveis": 1
          }));
          preencherFinanceira();
          gerarGrafico();
        } else {
          alert("Formato de arquivo não reconhecido.");
        }
      }
    });
  });

  function preencherFiltros() {
    const regioes = [...new Set(dadosVeiculos.map(v => v.Região))].filter(Boolean);
    const filtro = document.getElementById("filtroRegiao");
    filtro.innerHTML = <option value="">Todas</option>;
    regioes.forEach(reg => {
      filtro.innerHTML += <option value="${reg}">${reg}</option>;
    });
  }

  function aplicarFiltros() {
    const regiao = document.getElementById("filtroRegiao").value;
    const situacao = document.getElementById("filtroSituacao").value;
    const dataInicio = document.getElementById("filtroDataInicio").value;
    const dataFim = document.getElementById("filtroDataFim").value;

    dadosFiltrados = dadosVeiculos.filter(v => {
      const dentroRegiao = !regiao || v.Região === regiao;
      const dentroSituacao = !situacao || v.Situação === situacao;
      const data = new Date(v.Data.split('/').reverse().join('-'));
      const dentroData = (!dataInicio || data >= new Date(dataInicio)) &&
                         (!dataFim || data <= new Date(dataFim));
      return dentroRegiao && dentroSituacao && dentroData;
    });
    preencherTabelaVeiculos();
  }

  function preencherTabelaVeiculos() {
    const tbody = document.querySelector("#tabelaVeiculos tbody");
    tbody.innerHTML = "";
    (dadosFiltrados.length ? dadosFiltrados : dadosVeiculos).forEach(v => {
      tbody.innerHTML += `<tr>
        <td>${v.Data}</td><td>${v.Mapa}</td><td>${v["Hora Saída"]}</td><td>${v.Região}</td><td>${v.Situação}</td>
      </tr>`;
    });
  }

  function preencherFinanceira() {
    const tbody = document.querySelector("#tabelaFinanceira tbody");
    tbody.innerHTML = "";
    const resumo = {};
    dadosFinanceiros.forEach(item => {
      if (!resumo[item.Segmento]) resumo[item.Segmento] = { desconto: 0, dias: 0 };
      resumo[item.Segmento].desconto += item["Desconto FA (R$)"];
      resumo[item.Segmento].dias += item["Dias Disponíveis"];
    });

    let totalDesconto = 0, totalDias = 0;
    for (const seg in resumo) {
      totalDesconto += resumo[seg].desconto;
      totalDias += resumo[seg].dias;
      tbody.innerHTML += `<tr>
        <td>${seg}</td><td>R$ ${resumo[seg].desconto.toFixed(2)}</td><td>${resumo[seg].dias}</td>
      </tr>`;
    }

    tbody.innerHTML += `<tr style="font-weight:bold">
      <td>Total Unificado</td><td>R$ ${totalDesconto.toFixed(2)}</td><td>${totalDias}</td>
    </tr>`;
  }

  function gerarGrafico() {
    const ctx = document.getElementById('graficoDescontos').getContext('2d');
    if (window.grafico) window.grafico.destroy();
    const labels = dadosFinanceiros.map(d => d.Segmento);
    const valores = dadosFinanceiros.map(d => d["Desconto FA (R$)"]);
    window.grafico = new Chart(ctx, {
      type: 'bar',
      data: {
        labels,
        datasets: [{
          label: 'Desconto (R$)',
          data: valores,
          backgroundColor: ['#4e73df', '#1cc88a', '#f6c23e', '#e74a3b']
        }]
      },
      options: {
        responsive: true,
        plugins: {
          legend: { display: false }
        },
        scales: {
          y: {
            beginAtZero: true,
            ticks: { callback: v => R$ ${v} }
          }
        }
      }
    });
  }

  function gerarDadosExemplo() {
    dadosVeiculos = [
      { Data: "2025-06-23", Mapa: "921607", "Hora Saída": "19:05", Região: "CENTRO", Situação: "Roteirizado" },
      { Data: "2025-06-23", Mapa: "921608", "Hora Saída": "19:06", Região: "FEDERAÇÃO", Situação: "Roteirizado" }
    ];
    dadosFinanceiros = [
      { Segmento: "VAN", "Desconto FA (R$)": 2567.43, "Dias Disponíveis": 20 },
      { Segmento: "PADRÃO", "Desconto FA (R$)": 7983.55, "Dias Disponíveis": 22 }
    ];
    preencherFiltros();
    aplicarFiltros();
    preencherFinanceira();
    gerarGrafico();
  }

  function exportarPDF() {
    const { jsPDF } = window.jspdf;
    const pdf = new jsPDF();
    pdf.text("Painel Logístico - Relatório", 10, 10);
    pdf.html(document.body, {
      callback: function (doc) {
        doc.save("painel-logistico.pdf");
      },
      x: 10,
      y: 20
    });
  }
</script>

</body>
</html>
