# Obsidian-RISM-fetcher
Script for the Obsidian Templater Plugin.
Fetches composer entries from the RISM Api and saves them in your current Markdown File as Metadata and Text. Fields in German. Feel free to adjust it to your needs.

![gif](https://github.com/user-attachments/assets/adfa6916-cb4a-454f-ae78-ca62a30b7b74)


```
<%*
// 1. Name abfragen
const initialName = tp.file.title.replace(/_/g, " ");
const searchName = await tp.system.prompt("RISM-Suche: Name des Komponisten?", initialName);

if (!searchName) {
    new Notice("Abgebrochen");
    return;
}

// Hilfsfunktion: Datum in ISO-Format umwandeln
const parseToISO = (dateStr) => {
    if (!dateStr) return "";
    const cleanStr = dateStr.trim();
    const match = cleanStr.match(/(\d{1,2})\.(\d{1,2})\.(\d{4})/);
    if (match) {
        const [_, d, m, y] = match;
        return `${y}-${m.padStart(2, '0')}-${d.padStart(2, '0')}`;
    }
    const yearMatch = cleanStr.match(/(\d{4})/);
    return yearMatch ? yearMatch[1] : cleanStr;
};

// 2. RISM API Suche
const response = await fetch(`https://rism.online/search?mode=people&q=${encodeURIComponent(searchName)}`, {
    headers: { 'Accept': 'application/ld+json' }
});
const searchData = await response.json();

if (!searchData.items || searchData.items.length === 0) {
    new Notice("Kein Eintrag gefunden.");
    return;
}

// 3. Auswahl des Eintrags
const selected = await tp.system.suggester(
    (item) => `${item.label.none[0]} (ID: ${item.id.split('/').pop()})`,
    searchData.items.slice(0, 10)
);

if (!selected) return;
const rid = selected.id.split('/').pop();

// 4. Details abrufen
const detailRes = await fetch(`https://rism.online/people/${rid}`, {
    headers: { 'Accept': 'application/ld+json', 'X-API-Accept-Language': 'de' }
});
const d = await detailRes.json();

// 5. DATEN PARSEN
let updates = {};

// A) Biografische Details für die Properties
const summary = d.biographicalDetails?.summary || [];
summary.forEach(i => {
    const label = i.label?.de?.[0] || i.label?.en?.[0];
    const val = i.value?.de || i.value?.none || i.value?.en || [];
    
    if (label && val.length > 0) {
        if (label.match(/Lebensdaten|Andere Lebensdaten|Geburts- und Todesdaten/)) {
            const parts = val[0].split("-");
            if (parts.length === 2) {
                updates["Geburtsdatum"] = parseToISO(parts[0]);
                updates["Sterbedatum"] = parseToISO(parts[1]);
            }
        } else {
            updates[label] = val.length === 1 ? val[0] : val;
        }
    }
});

// B) Beziehungen (Lehrer, Schüler, etc.)
if (d.relationships?.items) {
    d.relationships.items.forEach(item => {
        const role = item.role?.label?.de?.[0] || item.role?.label?.en?.[0];
        const target = item.relatedTo?.label?.none?.[0];
        if (role && target) {
            if (!updates[role]) updates[role] = [];
            if (typeof updates[role] === 'string') updates[role] = [updates[role]];
            if (!updates[role].includes(target)) updates[role].push(target);
        }
    });
}

// C) Normdaten & IDs
d.externalAuthorities?.items?.forEach(auth => {
    const label = auth.label?.none?.[0];
    if (label && label.includes(":")) {
        const [key, val] = label.split(":").map(s => s.trim());
        updates[key] = key.includes("Istituto Centrale") ? val : (auth.url || val);
    }
});

// 6. DATEI AKTUALISIEREN
const activeFile = app.workspace.getActiveFile();
if (activeFile) {
    // Frontmatter (Properties) aktualisieren
    await app.fileManager.processFrontMatter(activeFile, (fm) => {
        fm["RISM_ID"] = rid;
        for (const [key, value] of Object.entries(updates)) {
            fm[key] = value;
        }
    });

    // --- TEXT-BLOCK GENERIEREN (Body) ---
    let bodyAppend = "";

    // 1. Werkkataloge (mit Link)
    if (d.works?.worksCatalogs?.items) {
        const cats = d.works.worksCatalogs.items.map(i => {
            const title = i.label?.none?.[0] || i.label?.de?.[0] || "Katalog";
            return `- [${title}](${i.id})`;
        }).join("\n");
        if (cats) bodyAppend += `\n\n## Werkkatalog\n${cats}`;
    }

    // 2. Werk-Referenzen
    if (d.works?.workReferences?.items) {
        const refs = d.works.workReferences.items.map(i => `- ${i.value} [GND](${i.url})`).join("\n");
        if (refs) bodyAppend += `\n\n## Werk-Referenzen\n${refs}`;
    }

    // 3. Links
    const bodyLinks = d.externalResources?.items?.map(r => `- [${r.label?.none?.[0] || r.url}](${r.url})`).join("\n") || "";
    if (bodyLinks) bodyAppend += `\n\n## Links\n${bodyLinks}`;

    // 4. Literatur & Bemerkungen
    const lit = d.notes?.notes?.flatMap(n => n.value?.none || []).map(l => `- ${l.replace(/<[^>]*>/g, '')}`).join("\n") || "";
    if (lit) bodyAppend += `\n\n## RISM Literatur\n${lit}`;

    // Text in die Datei schreiben (nur wenn Sektionen nicht existieren)
    if (bodyAppend) {
        await app.vault.process(activeFile, (data) => {
            if (!data.includes("## Links") && !data.includes("## RISM Literatur") && !data.includes("## Werkkatalog")) {
                return data + bodyAppend;
            }
            return data;
        });
    }
    new Notice("RISM Daten inklusive Werkkatalog & Links geladen!");
}
%>
```
