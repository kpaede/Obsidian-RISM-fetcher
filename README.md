# Obsidian-RISM-fetcher
fetches composer entries with an Obsidian Template. Fields in German. Feed free to adjust it to your needs.

```
<%*
// 1. Name abfragen
const initialName = tp.file.title.replace(/_/g, " ");
const searchName = await tp.system.prompt("RISM-Suche: Name des Komponisten?", initialName);

if (!searchName) {
    new Notice("Abgebrochen");
    return;
}

// Hilfsfunktion: Datum in ISO-Format umwandeln (analog zum Python-Skript)
const parseToISO = (dateStr) => {
    if (!dateStr) return "";
    const cleanStr = dateStr.trim();
    // Versuche DD.MM.YYYY
    const match = cleanStr.match(/(\d{1,2})\.(\d{1,2})\.(\d{4})/);
    if (match) {
        const [_, d, m, y] = match;
        return `${y}-${m.padStart(2, '0')}-${d.padStart(2, '0')}`;
    }
    // Versuche nur das Jahr YYYY
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

// 3. Auswahl
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

// 5. Daten parsen
let updates = {};

// Biografische Details (Orte & Daten)
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
            // Dies fängt Geburtsort, Sterbeort, Wirkungsort automatisch ein
            updates[label] = val.length === 1 ? val[0] : val;
        }
    }
});

// Namensvarianten
const variants = d.nameVariants?.items?.flatMap(v => v.value?.none || []) || [];
if (variants.length) updates["Namensvarianten"] = variants;

// Beziehungen (Lehrer, Schüler, etc.)
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

// Normdaten & IDs
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
    await app.fileManager.processFrontMatter(activeFile, (fm) => {
        fm["RISM_ID"] = rid;
        for (const [key, value] of Object.entries(updates)) {
            fm[key] = value;
        }
    });

    // Links & Literatur
    const bodyLinks = d.externalResources?.items?.map(r => `- [${r.label?.none?.[0] || r.url}](${r.url})`).join("\n") || "";
    const lit = d.notes?.notes?.flatMap(n => n.value?.none || []).map(l => `- ${l.replace(/<[^>]*>/g, '')}`).join("\n") || "";

    let bodyAppend = "";
    if (bodyLinks) bodyAppend += `\n\n## Links\n${bodyLinks}`;
    if (lit) bodyAppend += `\n\n## RISM Literatur\n${lit}`;

    if (bodyAppend) {
        await app.vault.process(activeFile, (data) => {
            if (!data.includes("## Links") && !data.includes("## RISM Literatur")) {
                return data + bodyAppend;
            }
            return data;
        });
    }
    new Notice("RISM Daten erfolgreich geladen!");
}
%>
```
