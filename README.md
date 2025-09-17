<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>CSAF Report Viewer</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background: #f4f6f8; }
    h1 { color: #222; }
    .section { margin-bottom: 25px; padding: 20px; background: #fff; border-radius: 8px; box-shadow: 0 1px 4px rgba(0,0,0,0.1);}
    h2 { border-bottom: 2px solid #eee; padding-bottom: 5px; margin-bottom: 15px; }
    h3 { margin-top: 20px; }
    .meta { font-size: 0.9em; color: #444; margin-bottom: 10px; }
    .vuln { margin-bottom: 25px; }
    table { border-collapse: collapse; margin-top: 10px; }
    td, th { border: 1px solid #ccc; padding: 6px 10px; }
    th { background: #eee; }
    ul { padding-left: 18px; }
    button { padding: 8px 15px; margin-top: 10px; cursor: pointer; }
  </style>
</head>
<body>
  <h1>CSAF Report Viewer</h1>

  <div class="section">
    <h2>Upload CSAF JSON</h2>
    <input type="file" id="fileInput" accept=".json" multiple>
    <button onclick="processFiles()">Generate Reports</button>
    <button onclick="downloadCombined()">Download Combined CSAF</button>
  </div>

  <div id="reports"></div>

  <script>
    let allDocuments = [];

    async function processFiles() {
      const files = document.getElementById('fileInput').files;
      allDocuments = [];
      document.getElementById("reports").innerHTML = "";

      for (let file of files) {
        const text = await file.text();
        try {
          const json = JSON.parse(text);
          allDocuments.push(json);
          renderReport(json);
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

    function getNote(notes, category) {
      if (!notes) return null;
      const n = notes.find(n => n.category === category);
      return n ? n.text : null;
    }

    function renderReport(doc) {
      const container = document.getElementById("reports");
      const d = doc.document || {};
      const tracking = d.tracking || {};
      const publisher = d.publisher || {};

      let html = `<div class="section">
        <h2>${d.title || "Untitled Advisory"}</h2>
        <div class="meta">
          <b>Publisher:</b> ${publisher.name || "Unknown"} &nbsp; 
          <b>Category:</b> ${d.category || ""}<br>
          <b>Initial release:</b> ${tracking.initial_release_date || ""} &nbsp;
          <b>Current release:</b> ${tracking.current_release_date || ""}<br>
          <b>Status:</b> ${tracking.status || ""} &nbsp; 
          <b>Version:</b> ${tracking.version || ""}<br>
          <b>Engine:</b> ${tracking.generator?.engine?.name || ""} ${tracking.generator?.engine?.version || ""}
        </div>`;

      // Notes
      const summary = getNote(d.notes, "summary");
      if (summary) html += `<h3>Summary</h3><p>${summary}</p>`;

      const general = getNote(d.notes, "general");
      if (general) html += `<h3>General Recommendations</h3><p>${general}</p>`;

      const legal = getNote(d.notes, "legal_disclaimer");
      if (legal) html += `<h3>Disclaimer</h3><p>${legal}</p>`;

      // Vulnerabilities
      const vulns = doc.vulnerabilities || [];
      if (vulns.length) {
        html += `<h3>Vulnerabilities</h3>`;
        for (let v of vulns) {
          html += `<div class="vuln">
            <b>${v.cve || "No CVE"}:</b> ${v.title || ""}<br>
            <b>CWE:</b> ${v.cwe?.id || ""}: ${v.cwe?.name || ""}<br>`;

          // Vulnerability summary note
          const vsummary = getNote(v.notes, "summary");
