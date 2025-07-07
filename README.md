<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Inventario SYT</title>
  <style>
    body { font-family: Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 20px; }
    h1, h2 { color: #333; }
    input, button, textarea, select { padding: 6px; margin: 4px 0; width: 100%; box-sizing: border-box; }
    button { background: #fdd835; border: none; color: #000; cursor: pointer; }
    button:hover { background: #fbc02d; }
    .form-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; margin-top: 20px; }
    .form-grid button { grid-column: span 2; }
    .img-preview { display: flex; flex-wrap: wrap; gap: 5px; margin-top: 5px; }
    .img-preview img { width: 60px; height: 60px; object-fit: cover; border: 1px solid #ccc; border-radius: 4px; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; background: white; }
    th, td { border: 1px solid #ccc; padding: 8px; font-size: 14px; vertical-align: top; }
    th { background: #fdd835; }
    tr:nth-child(even) { background: #f9f9f9; }
    .acciones { margin-top: 20px; }
  </style>
</head>
<body>
  <h1>Control de Inventario SYT</h1>

  <input type="text" id="busqueda" placeholder="Buscar por descripción o Código SAP" />

  <div class="form-grid" id="formulario"></div>

  <input type="file" id="imagenes" accept="image/*" capture="environment" multiple />
  <div id="img-preview" class="img-preview"></div>
  <button onclick="guardarProducto()">Guardar producto (Entrada)</button>

  <div class="acciones">
    <h2>Movimientos de Material</h2>
    <button onclick="abrirEntrega()">Entrega de Material</button>
    <button onclick="abrirReintegro()">Reintegro de Material</button>
  </div>

  <table id="tabla">
    <thead><tr id="encabezados"></tr></thead>
    <tbody id="cuerpo"></tbody>
  </table>

  <h2>Historial de movimientos</h2>
  <ul id="historial"></ul>

  <!-- Modales -->
  <div id="modalEntrega" style="display:none; background:#fff; border:1px solid #ccc; padding:10px;">
    <h3>Entrega de Material</h3>
    <input type="text" id="entregaCodigo" placeholder="Código SAP" />
    <input type="number" id="entregaCantidad" placeholder="Cantidad a entregar" />
    <button onclick="procesarEntrega()">Confirmar Entrega</button>
    <button onclick="cerrarModal('modalEntrega')">Cancelar</button>
  </div>

  <div id="modalReintegro" style="display:none; background:#fff; border:1px solid #ccc; padding:10px;">
    <h3>Reintegro de Material</h3>
    <input type="text" id="reintegroCodigo" placeholder="Código SAP" />
    <input type="number" id="reintegroCantidad" placeholder="Cantidad a reintegrar" />
    <button onclick="procesarReintegro()">Confirmar Reintegro</button>
    <button onclick="cerrarModal('modalReintegro')">Cancelar</button>
  </div>

  <script>
    const campos = [
      "marcaTemporal", "codigoSAP", "reserva", "diciplina", "descripcion", "cantidad",
      "diametro", "sch", "longitud", "peso", "fabricante", "serial", "modelo",
      "escala", "certificado", "proveedor", "inspeccion", "observaciones"
    ];
    let productos = [];
    let imagenesBase64 = [];
    let historialMovimientos = [];

    const formulario = document.getElementById("formulario");
    const inputImagenes = document.getElementById("imagenes");
    const imgPreview = document.getElementById("img-preview");
    const cuerpo = document.getElementById("cuerpo");
    const encabezados = document.getElementById("encabezados");
    const historial = document.getElementById("historial");
    const busqueda = document.getElementById("busqueda");

    const WEBHOOK_URL = "https://script.google.com/macros/s/AKfycbwTqN1TdhbLN8mIX_MfpXK6_9dQ6wEY2L-AzmruaVcdlVHZFD8pqOdeHNrcwWVs7LHm7Q/exec";

    // Crear formulario dinámico
    campos.forEach(campo => {
      const input = document.createElement("input");
      input.placeholder = campo;
      input.name = campo;
      formulario.appendChild(input);
    });

    // Manejo de imágenes
    inputImagenes.addEventListener("change", (event) => {
      const files = Array.from(event.target.files).slice(0, 6);
      imagenesBase64 = [];
      imgPreview.innerHTML = "";
      files.forEach(file => {
        const reader = new FileReader();
        reader.onload = e => {
          imagenesBase64.push(e.target.result);
          const img = document.createElement("img");
          img.src = e.target.result;
          imgPreview.appendChild(img);
        };
        reader.readAsDataURL(file);
      });
    });

    async function guardarProducto() {
      const nuevo = {};
      campos.forEach(c => nuevo[c] = formulario.querySelector(`[name=${c}]`).value);
      nuevo.cantidad = parseInt(nuevo.cantidad) || 0;
      nuevo.imagenes = imagenesBase64.slice();
      nuevo.marcaTemporal = new Date().toLocaleString();

      try {
        const response = await fetch(WEBHOOK_URL, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(nuevo)
        });
        const res = await response.json();
        if (res.status === "success") {
          productos.push(nuevo);
          historialMovimientos.push(`[${nuevo.marcaTemporal}] Entrada: ${nuevo.descripcion} (${nuevo.codigoSAP}) x${nuevo.cantidad}`);
          limpiarFormulario();
          render();
          alert("Producto guardado correctamente en Google Sheets.");
        } else {
          alert("Error en servidor: " + res.message);
        }
      } catch (error) {
        console.error("Error al enviar:", error);
        alert("Error al guardar. Verifica conexión o permisos del webhook.");
      }
    }

    function abrirEntrega() {
      document.getElementById("modalEntrega").style.display = "block";
    }
    function abrirReintegro() {
      document.getElementById("modalReintegro").style.display = "block";
    }
    function cerrarModal(id) {
      document.getElementById(id).style.display = "none";
    }

    function procesarEntrega() {
      const codigo = document.getElementById("entregaCodigo").value;
      const cantidad = parseInt(document.getElementById("entregaCantidad").value);
      const producto = productos.find(p => p.codigoSAP === codigo);
      if (producto && producto.cantidad >= cantidad) {
        producto.cantidad -= cantidad;
        historialMovimientos.push(`[${new Date().toLocaleString()}] Entrega: ${producto.descripcion} (${codigo}) -${cantidad}`);
        render();
        cerrarModal('modalEntrega');
      } else {
        alert("Producto no encontrado o cantidad insuficiente.");
      }
    }

    function procesarReintegro() {
      const codigo = document.getElementById("reintegroCodigo").value;
      const cantidad = parseInt(document.getElementById("reintegroCantidad").value);
      const producto = productos.find(p => p.codigoSAP === codigo);
      if (producto) {
        producto.cantidad += cantidad;
        historialMovimientos.push(`[${new Date().toLocaleString()}] Reintegro: ${producto.descripcion} (${codigo}) +${cantidad}`);
        render();
        cerrarModal('modalReintegro');
      } else {
        alert("Producto no encontrado.");
      }
    }

    function limpiarFormulario() {
      formulario.querySelectorAll("input").forEach(input => input.value = "");
      inputImagenes.value = "";
      imgPreview.innerHTML = "";
      imagenesBase64 = [];
    }

    function render() {
      encabezados.innerHTML = campos.map(c => `<th>${c}</th>`).join("") + "<th>Imágenes</th>";
      cuerpo.innerHTML = productos.map(p => `
        <tr>
          ${campos.map(c => `<td>${p[c]}</td>`).join("")}
          <td>${(p.imagenes || []).map(src => `<img src='${src}' width='60' />`).join(" ")}</td>
        </tr>
      `).join("");
      historial.innerHTML = historialMovimientos.map(item => `<li>${item}</li>`).join("");
    }

    busqueda.addEventListener("input", () => {
      const filtro = busqueda.value.toLowerCase();
      const filas = cuerpo.querySelectorAll("tr");
      filas.forEach(fila => {
        const texto = fila.innerText.toLowerCase();
        fila.style.display = texto.includes(filtro) ? "" : "none";
      });
    });
  </script>
</body>
</html>
