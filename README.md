<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Escáner de códigos de barras</title>

  <!-- Biblioteca para leer códigos -->
  <script src="https://unpkg.com/html5-qrcode"></script>

  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      padding: 20px;
      font-family: Arial, sans-serif;
      background: #eef2ff;
      color: #172554;
    }

    .contenedor {
      max-width: 650px;
      margin: 25px auto;
      padding: 25px;
      background: white;
      border-radius: 18px;
      box-shadow: 0 8px 25px #00000015;
    }

    h1, h2 {
      text-align: center;
    }

    h1 {
      color: #4338ca;
      font-size: 27px;
    }

    .descripcion {
      text-align: center;
      color: #64748b;
    }

    #reader {
      width: 100%;
      margin: 20px auto;
    }

    .resultado {
      padding: 16px;
      background: #eef2ff;
      border-radius: 12px;
      margin: 20px 0;
      overflow-wrap: anywhere;
    }

    #codigo {
      font-size: 21px;
      font-weight: bold;
      color: #4338ca;
    }

    input {
      width: 100%;
      padding: 12px;
      margin: 8px 0;
      border: 1px solid #cbd5e1;
      border-radius: 8px;
      font-size: 16px;
    }

    button {
      border: none;
      border-radius: 8px;
      padding: 12px 16px;
      margin: 4px 2px;
      background: #4338ca;
      color: white;
      font-size: 15px;
      cursor: pointer;
    }

    button:hover {
      background: #312e81;
    }

    .secundario {
      background: #475569;
    }

    ul {
      padding-left: 22px;
    }

    li {
      margin: 12px 0;
      overflow-wrap: anywhere;
    }

    .aviso {
      font-size: 13px;
      color: #64748b;
    }
  </style>
</head>

<body>
  <main class="contenedor">
    <h1>📦 Escáner de códigos</h1>

    <p class="descripcion">
      Escanea un producto con tu cámara
      y consulta el código detectado.
    </p>

    <div id="reader"></div>

    <section class="resultado">
      <h2>Última lectura</h2>
      <p id="codigo">Aún no hay códigos</p>
      <p id="tipo"></p>

      <button id="copiar">Copiar código</button>
    </section>

    <h2>Agregar manualmente</h2>

    <input
      id="manual"
      type="text"
      placeholder="Escribe un código de barras"
    >

    <button id="agregar">Agregar código</button>

    <h2>Historial</h2>

    <p id="contador">Lecturas: 0</p>
    <ul id="historial"></ul>

    <button id="exportar">Exportar CSV</button>
    <button id="borrar" class="secundario">
      Borrar historial
    </button>

    <p class="aviso">
      Los datos se guardan localmente en este navegador.
    </p>
  </main>

  <script>
    const codigoElemento = document.getElementById("codigo");
    const tipoElemento = document.getElementById("tipo");
    const historialElemento = document.getElementById("historial");
    const contadorElemento = document.getElementById("contador");
    const manual = document.getElementById("manual");

    let historial = [];

    try {
      historial = JSON.parse(
        localStorage.getItem("historialCodigos") || "[]"
      );

      if (!Array.isArray(historial)) historial = [];
    } catch {
      historial = [];
    }

    let ultimoCodigo = "";
    let ultimaLectura = 0;

    function guardarHistorial() {
      try {
        localStorage.setItem(
          "historialCodigos",
          JSON.stringify(historial)
        );
      } catch {
        alert("No se pudo guardar el historial.");
      }
    }

    function agregarCodigo(valor, tipo = "Manual") {
      valor = String(valor).trim();

      if (!valor) return;

      codigoElemento.textContent = valor;
      tipoElemento.textContent = "Tipo: " + tipo;

      // Evitar registrar muchas veces el mismo
      // código durante un escaneo continuo.
      const ahora = Date.now();

      if (
        valor === ultimoCodigo &&
        ahora - ultimaLectura < 2000
      ) {
        return;
      }

      ultimoCodigo = valor;
      ultimaLectura = ahora;

      historial.unshift({
        codigo: valor,
        tipo: tipo,
        fecha: new Date().toLocaleString("es-MX")
      });

      dibujarHistorial();
      guardarHistorial();
    }

    function dibujarHistorial() {
      historialElemento.replaceChildren();

      historial.forEach((item) => {
        const li = document.createElement("li");
        const texto = document.createElement("span");

        texto.textContent =
          item.codigo + " — " + item.fecha +
          " (" + item.tipo + ")";

        li.appendChild(texto);
        historialElemento.appendChild(li);
      });

      contadorElemento.textContent =
        "Lecturas: " + historial.length;
    }

    // Escáner de cámara
    const escaner = new Html5QrcodeScanner(
      "reader",
      {
        fps: 10,
        qrbox: { width: 280, height: 130 },
        formatsToSupport: [
          Html5QrcodeSupportedFormats.EAN_13,
          Html5QrcodeSupportedFormats.EAN_8,
          Html5QrcodeSupportedFormats.UPC_A,
          Html5QrcodeSupportedFormats.UPC_E,
          Html5QrcodeSupportedFormats.CODE_128,
          Html5QrcodeSupportedFormats.CODE_39,
          Html5QrcodeSupportedFormats.ITF,
          Html5QrcodeSupportedFormats.QR_CODE
        ]
      },
      false
    );

    escaner.render(
      (texto, resultado) => {
        const formato =
          resultado?.result?.format?.formatName || "Código";

        agregarCodigo(texto, formato);
      },
      () => {
        // No hacer nada mientras busca un código.
      }
    );

    document.getElementById("agregar").addEventListener(
      "click",
      () => {
        agregarCodigo(manual.value, "Manual");
        manual.value = "";
      }
    );

    manual.addEventListener("keydown", (evento) => {
      if (evento.key === "Enter") {
        document.getElementById("agregar").click();
      }
    });

    document.getElementById("copiar").addEventListener(
      "click",
      async () => {
        const valor = codigoElemento.textContent;

        if (valor === "Aún no hay códigos") return;

        try {
          await navigator.clipboard.writeText(valor);
          alert("Código copiado.");
        } catch {
          alert("No se pudo copiar. Selecciona el código manualmente.");
        }
      }
    );

    document.getElementById("borrar").addEventListener(
      "click",
      () => {
        if (!confirm("¿Borrar todo el historial?")) return;

        historial = [];
        guardarHistorial();
        dibujarHistorial();
      }
    );

    document.getElementById("exportar").addEventListener(
      "click",
      () => {
        if (historial.length === 0) {
          alert("No hay lecturas para exportar.");
          return;
        }

        const filas = [
          ["Código", "Tipo", "Fecha"],
          ...historial.map(item => [
            item.codigo,
            item.tipo,
            item.fecha
          ])
        ];

        const csv = filas.map(fila =>
          fila.map(valor =>
            '"' + String(valor).replace(/"/g, '""') + '"'
          ).join(",")
        ).join("\r\n");

        const archivo = new Blob(
          ["\uFEFF" + csv],
          { type: "text/csv;charset=utf-8;" }
        );

        const url = URL.createObjectURL(archivo);
        const enlace = document.createElement("a");

        enlace.href = url;
        enlace.download = "historial-codigos.csv";
        enlace.click();

        URL.revokeObjectURL(url);
      }
    );

    dibujarHistorial();
  </script>
</body>
</html>
