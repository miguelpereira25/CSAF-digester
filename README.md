<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CSAF Visualizer & Merger</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background: #f9f9f9; }
    h1 { color: #333; }
    .section { margin-bottom: 20px; padding: 15px; background: #fff; border-radius: 8px; box-shadow: 0 1px 4px rgba(0,0,0,0.1);}
    ul { margin: 0; padding-left: 20px; }
    li { margin-bottom: 5px; }
    .vuln { margin-bottom: 15px; }
    button { padding: 8px 15px; margin-top: 10px; cursor: pointer; }
  </style>
</head>
<body>
  <h1>CSAF Visualizer & Merger</h1>

  <div class="section">
    <h2>Upload CSAF JSON(s)</h2>
    <input type="file" id="fileInput" accept=".json" multiple>
    <button onclick="processFiles()">Process Files</button>
    <button onclick="downloadCombined()">Download Combined CSAF</button>
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
      document.getElementById("output").innerHTML = "";

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

    function resolveProducts(doc, ids=[]) {
      if (!doc.product_tree || !doc.product_tree.full_product_names) return [];
      const map = {};
      for (let p of doc.product_tree.full_product_names) {
        map[p.product_id] = p.name;
      }
      return ids.map(id => map[id] || id);
    }

    function displayData(doc) {
      const out = document.getElementById('output');
      const docTitle = doc.document?.title || "Untitled";
      const publisher = doc.document?.publisher?.name || "Unknown";
      const vulnerabilities = doc.vulnerabilities || [];

      let html = `
        <h3>${docTitle}</h3>
        <p><b>Publisher:</b> ${publisher}</p>
        <p><b>Vulnerabilities:</b> ${vulnerabilities.length}</p>
      `;

      for (let v of vulnerabilities) {
        const affected = [];
        if (v.product_status) {
          for (let status of Object.keys(v.product_status)) {
            const products = resolveProducts(doc, v.product_status[status]);
            if (products.length) {
              affected.push(`<b>${status}:</b> ${products.join(", ")}`);
            }
          }
        }

        html += `
          <div class="vuln">
            <p><b>${v.cve || "No CVE"}:</b> ${v.title || ""} (${v.cwe?.id || ""})</p>
            <ul>${affected.map(a => `<li>${a}</li>`).join("")}</ul>
          </div>
        `;
      }

      out.innerHTML += html + "<hr/>";
    }

    function downloadCombined() {
      if (allDocuments.length === 0) {
        alert("No documents to combine!");
        return;
      }

      // Merge product trees and vulnerabilities
      let allProducts = [];
      let allVulns = [];
      for (let doc of allDocuments) {
        if (doc.product_tree?.full_product_names) {
          allProducts.push(...doc.product_tree.full_product_names);
        }
        if (doc.vulnerabilities) {
          allVulns.push(...doc.vulnerabilities);
        }
      }

      const combined = {
        document: {
          category: "csaf_security_advisory",
          csaf_version: "2.0",
          publisher: {
            category: "user",
            name: "CSAF Visualizer",
            namespace: "https://example.com"
          },
          title: "Combined CSAF Advisories",
          tracking: {
            id: "combined-" + Date.now(),
            status: "draft",
            version: "1",
            current_release_date: new Date().toISOString(),
            initial_release_date: new Date().toISOString(),
            revision_history: [
              { number: "1", date: new Date().toISOString(), summary: "Initial combined document" }
            ]
          }
        },
        product_tree: { full_product_names: dedup(allProducts, "product_id") },
        vulnerabilities: allVulns
      };

      const blob = new Blob([JSON.stringify(combined, null, 2)], {type: "application/json"});
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "combined_csaf.json";
      a.click();
      URL.revokeObjectURL(url);
    }

    function dedup(arr, key) {
      const seen = new Set();
      return arr.filter(item => {
        if (seen.has(item[key])) return false;
        seen.add(item[key]);
        return true;
      });
    }
  </script>
</body>
</html>
