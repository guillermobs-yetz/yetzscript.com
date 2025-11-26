<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Creador de Alfabeto Activable</title>
  <style>
    :root{--gap:10px}
    body{font-family:system-ui,Segoe UI,Roboto,Arial;margin:20px}
    h1{margin-bottom:5px}
    .controls{display:flex;gap:var(--gap);align-items:center;flex-wrap:wrap}
    .grid{display:flex;flex-wrap:wrap;gap:10px;margin-top:10px}
    .symbol-box{width:80px;height:80px;border:1px solid #444;display:flex;align-items:center;justify-content:center}
    canvas.symbol{width:80px;height:80px;background:#fff;cursor:crosshair}
    .map-row{display:flex;gap:8px;align-items:center}
    .output{min-height:120px;border:1px dashed #666;padding:8px;margin-top:12px}
    .button{padding:8px 12px;border:0;border-radius:6px;background:#0366d6;color:#fff;cursor:pointer}
    label.switch{display:inline-flex;align-items:center;gap:8px}
    .small{font-size:0.9rem}
  </style>
</head>
<body>
  <h1>Creador de Alfabeto activable</h1>
  <p class="small">Dibuja símbolos, asigna teclas y activa <strong>"Activar alfabeto"</strong> para escribir con tus símbolos en el recuadro editable. Si una tecla no tiene símbolo asignado, no escribirá nada.</p>

  <section>
    <div class="controls">
      <button id="addSymbol" class="button">Añadir símbolo</button>
      <button id="exportAll" class="button">Exportar alfabeto (JSON)</button>
      <input id="loadFile" type="file" accept="application/json" />
      <label class="switch"><input type="checkbox" id="toggleActive" /><span>Activar alfabeto</span></label>
      <button id="copyHtml" class="button">Copiar resultado (HTML)</button>
      <button id="copyPlain" class="button">Copiar texto plano</button>
    </div>

    <div class="grid" id="symbolGrid" aria-live="polite" style="margin-top:12px"></div>

    <h3>Asignaciones (Símbolo → Tecla)</h3>
    <div id="keyMapping"></div>

    <h3>Salida (editable). Pega o escribe aquí mientras el alfabeto está activado.</h3>
    <div id="output" class="output" contenteditable="true" aria-label="Salida editable"></div>
  </section>

  <script>
    // Estado
    const symbols = []; // {canvas, dataURL}
    const keyMap = new Map(); // key -> dataURL

    const symbolGrid = document.getElementById('symbolGrid');
    const keyMapping = document.getElementById('keyMapping');
    const addSymbolBtn = document.getElementById('addSymbol');
    const toggleActive = document.getElementById('toggleActive');
    const output = document.getElementById('output');
    const exportAll = document.getElementById('exportAll');
    const loadFile = document.getElementById('loadFile');
    const copyHtml = document.getElementById('copyHtml');
    const copyPlain = document.getElementById('copyPlain');

    // Crear un canvas dibujable
    function createSymbolCanvas(initialDataURL){
      const wrapper = document.createElement('div');
      wrapper.className = 'symbol-box';

      const canvas = document.createElement('canvas');
      canvas.width = 120; // más resolución interna
      canvas.height = 120;
      canvas.className = 'symbol';

      // escala CSS para mostrar 80x80
      canvas.style.width = '80px';
      canvas.style.height = '80px';

      const ctx = canvas.getContext('2d');
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(0,0,canvas.width,canvas.height);

      let drawing = false;
      let last = null;

      function getPos(e){
        const rect = canvas.getBoundingClientRect();
        const x = (e.clientX - rect.left) * (canvas.width / rect.width);
        const y = (e.clientY - rect.top) * (canvas.height / rect.height);
        return {x,y};
      }

      canvas.addEventListener('pointerdown',(e)=>{drawing=true; last=getPos(e)});
      canvas.addEventListener('pointerup',()=>{drawing=false; last=null; saveState(); updateKeyMapping();});
      canvas.addEventListener('pointermove',(e)=>{ if(!drawing) return; const p=getPos(e); ctx.strokeStyle='#000'; ctx.lineWidth=6; ctx.lineCap='round'; ctx.beginPath(); ctx.moveTo(last.x,last.y); ctx.lineTo(p.x,p.y); ctx.stroke(); last=p; });

      // botón borrar
      const clearBtn = document.createElement('button');
      clearBtn.className = 'button small';
      clearBtn.style.padding='6px';
      clearBtn.style.fontSize='12px';
      clearBtn.textContent = 'Borrar';
      clearBtn.addEventListener('click', ()=>{ctx.clearRect(0,0,canvas.width,canvas.height); ctx.fillStyle='#fff'; ctx.fillRect(0,0,canvas.width,canvas.height); saveState(); updateKeyMapping();});

      // cargar imagen inicial si existe
      if(initialDataURL){
        const img = new Image();
        img.onload = ()=>{ctx.drawImage(img,0,0,canvas.width,canvas.height); saveState(); updateKeyMapping();}
        img.src = initialDataURL;
      }

      wrapper.appendChild(canvas);
      wrapper.appendChild(clearBtn);
      return {wrapper, canvas};
    }

    function saveState(){
      // update symbols array dataURLs
      symbols.forEach(s=>{ s.dataURL = s.canvas.toDataURL('image/png'); });
    }

    function addSymbol(initialDataURL){
      const obj = createSymbolCanvas(initialDataURL);
      symbolGrid.appendChild(obj.wrapper);
      symbols.push({canvas: obj.canvas, dataURL: initialDataURL || obj.canvas.toDataURL('image/png')});
      updateKeyMapping();
    }

    // UI: actualización de asignaciones
    function updateKeyMapping(){
      // refresh dataURLs first
      saveState();
      keyMapping.innerHTML = '';
      symbols.forEach((s, idx)=>{
        const row = document.createElement('div');
        row.className = 'map-row';

        const img = document.createElement('img');
        img.src = s.dataURL || s.canvas.toDataURL();
        img.width = 40; img.height = 40;

        const label = document.createElement('span');
        label.textContent = `Símbolo ${idx+1}`;

        const select = document.createElement('input');
        select.placeholder = 'tecla (ej: a, 1, Enter)';
        select.value = '';
        select.style.width = '90px';

        // cargar desde keyMap si existe
        const existing = Array.from(keyMap.entries()).find(([k,v])=>v === s.dataURL);
        if(existing) select.value = existing[0];

        const saveBtn = document.createElement('button');
        saveBtn.className = 'button small';
        saveBtn.textContent = 'Asignar';
        saveBtn.addEventListener('click', ()=>{
          const key = select.value.trim().toLowerCase();
          if(!key){ alert('Introduce una tecla (ej: a, 1, enter)'); return; }
          // eliminar si otra tiene la misma key
          keyMap.forEach((val,k)=>{ if(k===key) keyMap.delete(k); });
          keyMap.set(key, s.dataURL);
          updateKeyMapping();
        });

        const removeMapping = document.createElement('button');
        removeMapping.className='button small';
        removeMapping.textContent='Quitar';
        removeMapping.addEventListener('click', ()=>{
          // quitar cualquier mapping que apunte a esta dataURL
          for(const [k,v] of keyMap.entries()){
            if(v===s.dataURL) keyMap.delete(k);
          }
          updateKeyMapping();
        });

        row.appendChild(img);
        row.appendChild(label);
        row.appendChild(select);
        row.appendChild(saveBtn);
        row.appendChild(removeMapping);
        keyMapping.appendChild(row);
      });

      // mostrar listado actual de asignaciones
      const list = document.createElement('div');
      list.style.marginTop='8px';
      list.innerHTML = '<strong>Asignaciones actuales:</strong> ' + (keyMap.size ? '' : '<em>ninguna</em>');
      for(const [k,v] of keyMap.entries()){
        const img = document.createElement('img'); img.src=v; img.width=20; img.height=20; img.style.verticalAlign='middle';
        const span = document.createElement('span'); span.style.marginRight='10px'; span.style.marginLeft='6px'; span.textContent = k;
        const entry = document.createElement('span'); entry.appendChild(img); entry.appendChild(span);
        list.appendChild(entry);
      }
      keyMapping.appendChild(list);
    }

    // Insertar imagen en caret de output
    function insertImageAtCaret(dataURL){
      const img = document.createElement('img');
      img.src = dataURL;
      img.style.width = '24px'; img.style.height='24px';
      const sel = window.getSelection();
      if(!sel || sel.rangeCount===0){ output.appendChild(img); return; }
      const range = sel.getRangeAt(0);
      range.deleteContents();
      range.insertNode(img);
      // mover caret después de la imagen
      range.setStartAfter(img);
      range.setEndAfter(img);
      sel.removeAllRanges();
      sel.addRange(range);
    }

    // Listener de teclado global
    function onKeyDown(e){
      if(!toggleActive.checked) return; // no activo
      // ignore when focus is on input/select/button other than output
      const active = document.activeElement;
      // allow typing in textarea/selects when they explicitly focus something other than output
      if(active && active !== document.body && active !== output){
        // if it's the output itself, we handle below
      }
      const key = e.key.toLowerCase();
      // Support for 'enter' mapping and others
      let mapped = keyMap.get(key);
      if(!mapped){
        // also try special names: 'enter', 'space', 'backspace'
        if(key === ' ' || key === 'space') mapped = keyMap.get('space') || null;
      }
      if(mapped){
        e.preventDefault();
        insertImageAtCaret(mapped);
      } else {
        // if no mapping, do nothing (i.e., prevent default only if the focused element is output)
        if(document.activeElement === output){
          e.preventDefault();
        }
      }
    }

    // Exports/Imports
    exportAll.addEventListener('click', ()=>{
      saveState();
      const dump = {symbols: symbols.map(s=>s.dataURL), mappings: Array.from(keyMap.entries())};
      const blob = new Blob([JSON.stringify(dump, null, 2)], {type:'application/json'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a'); a.href=url; a.download='alfabeto.json'; a.click(); URL.revokeObjectURL(url);
    });

    loadFile.addEventListener('change',(e)=>{
      const f = e.target.files[0]; if(!f) return;
      const reader = new FileReader();
      reader.onload = ()=>{
        try{
          const obj = JSON.parse(reader.result);
          // limpiar
          symbolGrid.innerHTML=''; symbols.length=0; keyMap.clear();
          (obj.symbols||[]).forEach(d=>addSymbol(d));
          (obj.mappings||[]).forEach(([k,v])=>keyMap.set(k,v));
          updateKeyMapping();
        }catch(err){ alert('Archivo no válido: '+err.message); }
      };
      reader.readAsText(f);
    });

    // Copiar resultados
    copyHtml.addEventListener('click', async ()=>{
      try{
        const html = output.innerHTML;
        const plaintext = output.innerText;
        // try modern clipboard with html
        if(navigator.clipboard && window.ClipboardItem){
          const blobHtml = new Blob([html], {type:'text/html'});
          const blobText = new Blob([plaintext], {type:'text/plain'});
          await navigator.clipboard.write([new ClipboardItem({'text/html': blobHtml, 'text/plain': blobText})]);
          alert('Copiado (HTML) — ahora puedes pegar en un editor que acepte contenido HTML (Gmail, Docs, etc.)');
        } else {
          // fallback: select and execCommand
          const range = document.createRange(); range.selectNodeContents(output); const sel = window.getSelection(); sel.removeAllRanges(); sel.addRange(range);
          document.execCommand('copy'); sel.removeAllRanges(); alert('Copiado (fallback)');
        }
      }catch(err){ alert('Error al copiar: '+err.message); }
    });

    copyPlain.addEventListener('click', async ()=>{
      try{
        const text = output.innerText;
        await navigator.clipboard.writeText(text);
        alert('Texto plano copiado al portapapeles');
      }catch(err){
        // fallback
        const range = document.createRange(); range.selectNodeContents(output); const sel = window.getSelection(); sel.removeAllRanges(); sel.addRange(range);
        document.execCommand('copy'); sel.removeAllRanges(); alert('Copiado (fallback)');
      }
    });

    // Añadir símbolo inicial de ejemplo
    addSymbolBtn.addEventListener('click', ()=>addSymbol());

    // Atajo para activar/desactivar con Alt+L
    document.addEventListener('keydown',(e)=>{ if(e.altKey && e.key.toLowerCase()==='l'){ toggleActive.checked = !toggleActive.checked; alert('Activar alfabeto: '+toggleActive.checked); } });

    // registrar listener global para interceptar teclas
    document.addEventListener('keydown', onKeyDown);

    // añadir primer símbolo por defecto
    addSymbol();
  </script>
</body>
</html>
