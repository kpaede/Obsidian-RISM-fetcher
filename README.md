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

// 2. RISM API Suche
const response = await fetch(`https://rism.online/search?mode=people&q=${encodeURIComponent(searchName)}`, {
    headers: { 'Accept': 'application/ld+json' }
});
const searchData = await response.json();

if (!searchData.items || searchData.items.length === 0) {
    new Notice("Kein Eintrag gefunden.");
    return;
}

// 3. Auswahl aus den Ergebnissen
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
const summary = d.biographicalDetails?.summary || [];
summary.forEach(i => {
    const label = i.label?.de?.[0];
    const val = i.value?.de || i.value?.none || i.value?.en || [];
    if (label) {
        if (label.includes("Lebensdaten") || label.includes("Geburts- und Todesdaten")) {
            const parts = val[0]?.split("-");
            if (parts?.length === 2) {
                updates["Geburtsdatum"] = parts[0].trim();
                updates["Sterbedatum"] = parts[1].trim();
            }
        } else {
            updates[label] = val.length === 1 ? val[0] : val;
        }
    }
});

const variants = d.nameVariants?.items?.flatMap(v => v.value?.none || []) || [];
if (variants.length) updates["Namensvarianten"] = variants;

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
    // Frontmatter verarbeiten
    await app.fileManager.processFrontMatter(activeFile, (fm) => {
        fm["RISM_ID"] = rid;
        for (const [key, value] of Object.entries(updates)) {
            fm[key] = value;
        }
    });

    // Body (Links & Literatur)
    const bodyLinks = d.externalResources?.items?.map(r => `- [${r.label?.none?.[0] || r.url}](${r.url})`).join("\n") || "";
    const lit = d.notes?.notes?.flatMap(n => n.value?.none || []).map(l => `- ${l.replace(/<[^>]*>/g, '')}`).join("\n") || "";

    let bodyAppend = "";
    if (bodyLinks) bodyAppend += `\n\n## Links\n${bodyLinks}`;
    if (lit) bodyAppend += `\n\n## RISM Literatur\n${lit}`;

    if (bodyAppend) {
        // Wir nutzen process(), um den Text sicher ans Ende zu hängen
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
