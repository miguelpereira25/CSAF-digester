<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>CSAF Visualizer</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- Styling -->
    <style>
      body { font-family: Arial, sans-serif; margin: 20px; background: #f9f9f9; }
      h1 { color: #333; }
      .section { margin-bottom: 20px; padding: 15px; background: #fff; border-radius: 8px; box-shadow: 0 1px 4px rgba(0,0,0,0.1);}
      pre { background: #272822; color: #f8f8f2; padding: 10px; overflow-x: auto; border-radius: 6px;}
      button { padding: 8px 15px; margin-top: 10px; cursor: pointer; }
      #output { white-space: pre-wrap; }
    </style>
  </head>
  <body>
    <h1>CSAF Visualizer</h1>
    <div class="section">
      <h2>Upload CSAF JSON(s)</h2>
      <input type="file" id="fileInput" accept=".json" multiple>
      <button onclick="processFiles()">Process Files</button>
      <button onclick="downloadCombined()">Download Combined JSON</button>
    </div>
    <div class="section">
      <h2>Extracted Data</h2>
      <div id="output">No data yet.</div>
    </div>
    <script>
      let allDocuments = [];
      async function processFiles() {
        const files = document.getElementById('fileInput').files;
        allDocuments = [];
        for (let file of files) {
          const text = await file.text();
          try {
            const json = JSON.parse(text);
            allDocuments.push(json);
            displayData(json);
          } catch (err) {
            alert(`Error parsing ${file.name}: ${err}`);
          }
        }
      }
      function displayData(doc) {
        const out = document.getElementById('output');
        const docTitle = doc.document?.title || "Untitled";
        const publisher = doc.document?.publisher?.name || "Unknown";
        const vulnerabilities = doc.vulnerabilities || [];
        out.innerHTML += `
          <h3>${docTitle}</h3>
          <p><b>Publisher:</b> ${publisher}</p>
          <p><b>Vulnerabilities:</b> ${vulnerabilities.length}</p>
          <ul>
            ${vulnerabilities.map(v => `
              <li><b>${v.cve || "No CVE"}:</b> ${v.title || ""} (${v.cwe?.id || ""})</li>
            `).join("")}
          </ul>
          <hr/>
        `;
      }
      function downloadCombined() {
        if (allDocuments.length === 0) {
          alert("No documents to combine!");
          return;
        }
        const combined = {
          combined_at: new Date().toISOString(),
          documents: allDocuments
        };
        const blob = new Blob([JSON.stringify(combined, null, 2)], {type: "application/json"});
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "combined_csaf.json";
        a.click();
        URL.revokeObjectURL(url);
      }
    </script>
  </body>
</html>
