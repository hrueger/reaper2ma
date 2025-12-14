<script lang="ts">
    import * as csv from "@vanillaes/csv";
    import { XMLBuilder } from "fast-xml-parser";

    let fileInput: HTMLInputElement;
    let uploadArea: HTMLElement;
    let sequenceNumber = 101;
    let driveNumber = 2;
    let prefix = "1";
    let exportMode: "cues-and-timecode" | "cues-only" = "cues-and-timecode";
    let isDragOver = false;
    let isProcessing = false;
    let processingStatus = "";
    let processingCompleted = false;

    type ParsedData = {
        name: string;
        start: string;
        color: string;
    }[];

    type RepeatedSequenceData = {
        color: string;
        name: string;
        timestamps: string[];
        sequenceNumber: number;
    }[];

    type FullData = {
        uniqueSequences: ParsedData;
        repeatedSequences: RepeatedSequenceData;
        filename: string;
    };

    function handleFileChange(event: Event) {
        const input = event.target as HTMLInputElement;
        if (input.files && input.files.length > 0) {
            const file = input.files[0];
            const reader = new FileReader();
            reader.onload = (e) => {
                const text = e.target?.result;
                const filename = file.name
                    .replace(".csv", "")
                    .toLowerCase()
                    .replace(/[^a-z]/g, "");
                processCsv(text as string, filename);
            };
            reader.readAsText(file);
        }
    }

    function downloadContentWithName(content: string, filename: string) {
        const blob = new Blob([content], { type: "application/xml" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }

    function safeName(name: string): string {
        return name.replace(/[^a-zA-Z0-9äöüÄÖÜß \-_#%\/\(\)\[\]=+]/g, "");
    }

    async function processCsv(dataString: string, filename: string) {
        isProcessing = true;
        processingStatus = "Parsing CSV data...";

        try {
            await new Promise((resolve) => setTimeout(resolve, 100)); // Small delay for UI

            const parsedLines = csv.parse(dataString) as string[][];
            const header = parsedLines[0];
            const dataByHeader = parsedLines.slice(1).map((row) => {
                const obj: Record<string, string> = {};
                row.forEach((value, index) => {
                    obj[header[index]] = value;
                });
                return obj;
            }) as { "#": string; Name: string; Start: string; Color: string }[];

            const seenNameCount: Record<string, number> = {};

            const data = dataByHeader
                .map((item) => {
                    let name = safeName(item.Name);
                    seenNameCount[name] = (seenNameCount[name] || 0) + 1;
                    return {
                        name: name,
                        start: item.Start,
                        color: item.Color,
                    };
                })
                .reverse()
                .map((item) => ({
                    ...item,
                    // add 1 for the first ones
                    name: seenNameCount[item.name] > 1 ? `${item.name} ${seenNameCount[item.name]--}` : item.name,
                }))
                .reverse();

            const uniqueSequences = data.filter((item) => !item.color);
            const repeatedSequenceList = data.filter((item) => item.color);

            let sqCounter = sequenceNumber + 1;
            const repeatedSequences: RepeatedSequenceData = repeatedSequenceList.reduce((acc, curr, index) => {
                const existing = acc.find((item) => item.color === curr.color);
                if (existing) {
                    existing.timestamps.push(curr.start);
                } else {
                    acc.push({
                        color: curr.color,
                        name: `${prefix} - ${curr.name}`,
                        timestamps: [curr.start],
                        sequenceNumber: sqCounter++,
                    });
                }
                return acc;
            }, [] as RepeatedSequenceData);

            processingStatus = "Generating XML files...";
            await new Promise((resolve) => setTimeout(resolve, 100));

            const xmlContent = generateMacroXML({ repeatedSequences, uniqueSequences, filename });
            downloadContentWithName(xmlContent, `${filename}_macro.xml`);

            if (exportMode === "cues-and-timecode") {
                const timecodeContent = generateTimecodeXML({ repeatedSequences, uniqueSequences, filename });
                downloadContentWithName(timecodeContent, `${filename}_timecode.xml`);
            }

            processingStatus = "✅ Files generated successfully!";
            setTimeout(() => {
                isProcessing = false;
                processingStatus = "";
                processingCompleted = true;
            }, 500);
        } catch (error) {
            processingStatus = "❌ Error processing file";
            setTimeout(() => {
                isProcessing = false;
                processingStatus = "";
            }, 3000);
            console.error("Error processing CSV:", error);
        }
    }

    // Drag and drop handlers
    function handleDragOver(event: DragEvent) {
        event.preventDefault();
        isDragOver = true;
    }

    function handleDragLeave(event: DragEvent) {
        event.preventDefault();
        if (!uploadArea.contains(event.relatedTarget as Node)) {
            isDragOver = false;
        }
    }

    function handleDrop(event: DragEvent) {
        event.preventDefault();
        isDragOver = false;

        const files = event.dataTransfer?.files;
        if (files && files.length > 0) {
            const file = files[0];
            if (file.type === "text/csv" || file.name.endsWith(".csv")) {
                // Simulate file input selection
                fileInput.files = files;
                handleFileChange({ target: fileInput } as any);
            } else {
                alert("Please select a CSV file");
            }
        }
    }

    function clearFile() {
        fileInput.value = "";
        processingCompleted = false;
        processingStatus = "";
        isProcessing = false;
    }

    const builder = new XMLBuilder({
        attributeNamePrefix: "@_",
        ignoreAttributes: false,
        format: true,
        suppressEmptyNode: true,
        indentBy: "    ",
    });
    const XML_HEADER = {
        "?xml": {
            "@_version": "1.0",
            "@_encoding": "UTF-8",
        },
    };

    function generateGUID() {
        return Array.from({ length: 16 }, () => Math.floor(Math.random() * 256))
            .map((b) => b.toString(16).padStart(2, "0").toUpperCase())
            .join(" ");
    }

    function generateMacroXML({ repeatedSequences, uniqueSequences, filename }: FullData): string {
        const obj = {
            ...XML_HEADER,
            GMA3: {
                "@_DataVersion": "1.4.0.2",
                Macro: {
                    "@_Name": `Macro ${filename}`,
                    "@_Guid": "00 00 00 00 A8 F8 B9 20 78 06 00 00 A5 46 09 AA",
                    MacroLine: [
                        // unique sequences
                        {
                            "@_Command": `Store Sequence ${sequenceNumber} Cue 1 thru ${uniqueSequences.length}`,
                            "@_Wait": "0.10",
                        },
                        ...uniqueSequences.map((item, index) => ({
                            "@_Command": `Label Sequence ${sequenceNumber} Cue ${index + 1} "${item.name}"`,
                            "@_Wait": "0.10",
                        })),
                        /// repeated sequences
                        ...repeatedSequences.flatMap((u, index) => {
                            return [
                                {
                                    "@_Command": `Store Sequence ${u.sequenceNumber} "${u.name}"`,
                                    "@_Wait": "0.10",
                                },
                                {
                                    "@_Command": `Store Cue 1 Sequence ${u.sequenceNumber} /Merge`,
                                    "@_Wait": "0.10",
                                },
                                {
                                    "@_Command": `Set Sequence ${u.sequenceNumber} Cue "OffCue" Property "TRIGTYPE" "Follow"`,
                                    "@_Wait": "0.10",
                                },
                            ];
                        }),
                        /// Timecode Import (only if exportMode is cues-and-timecode)
                        ...(exportMode === "cues-and-timecode"
                            ? [
                                  {
                                      "@_Command": `Drive ${driveNumber}`,
                                      "@_Wait": "0.10",
                                  },
                                  {
                                      "@_Command": `import Timecode "${filename}_timecode"`,
                                      "@_Wait": "0.10",
                                  },
                              ]
                            : []),
                    ],
                },
            },
        };

        return builder.build(obj);
    }

    function generateTimecodeXML({ uniqueSequences, repeatedSequences, filename }: FullData): string {
        const obj = {
            ...XML_HEADER,
            GMA3: {
                "@_DataVersion": "1.4.0.2",
                Timecode: {
                    "@_Name": filename,
                    "@_Guid": "00 00 00 00 3F 76 B7 04 32 0B 00 00 68 A1 4F AA",
                    "@_Cursor": "00.00",
                    "@_Duration": uniqueSequences.length > 0 ? (parseFloat(uniqueSequences[uniqueSequences.length - 1].start) + 1).toFixed(3) : "0.00",
                    "@_LoopCount": "0",
                    "@_TCSlot": "-1",
                    "@_SwitchOff": "Keep Playbacks",
                    "@_Goto": "as Go",
                    "@_Timedisplayformat": "<10d11h23m45>",
                    "@_FrameReadout": "<Seconds>",
                    TrackGroup: [
                        {
                            "@_Play": "",
                            "@_Rec": "",
                            MarkerTrack: {
                                "@_Name": "Marker",
                                "@_Guid": "00 00 00 00 B1 F5 25 5F 70 04 00 00 28 74 D0 4B",
                            },
                            Track: {
                                "@_Guid": "00 00 00 00 B7 10 04 13 3B 0B 00 00 38 E1 4F AA",
                                "@_Target": `ShowData.DataPools.Default.Sequences.Sequence ${sequenceNumber}`,
                                "@_Play": "",
                                "@_Rec": "",
                                TimeRange: {
                                    "@_Guid": "00 00 00 00 21 68 E5 6F 3C 0B 00 00 38 E1 4F AA",
                                    "@_Play": "",
                                    "@_Rec": "",
                                    CmdSubTrack: {
                                        CmdEvent: uniqueSequences.map((item, index) => ({
                                            "@_Name": "Go+",
                                            "@_Time": item.start,
                                            RealtimeCmd: {
                                                "@_Type": "Key",
                                                "@_Source": "Original",
                                                "@_UserProfile": "0",
                                                "@_Status": "On",
                                                "@_IsRealtime": "1",
                                                "@_IsXFade": "0",
                                                "@_IgnoreFollow": "0",
                                                "@_IgnoreCommand": "0",
                                                "@_Assert": "0",
                                                "@_IgnoreNetwork": "0",
                                                "@_FromTriggerNode": "0",
                                                "@_IgnoreExecTime": "0",
                                                "@_IssuedByTimecode": "0",
                                                "@_FromLocalHardwareFader": "1",
                                                ...(index == 0
                                                    ? {
                                                          "@_Object": `ShowData.DataPools.Default.Sequences.Sequence ${sequenceNumber}`,
                                                      }
                                                    : {}),
                                                "@_ExecToken": "Go+",
                                                "@_ValCueDestination": `ShowData.DataPools.Default.Sequences.Sequence ${sequenceNumber}.${item.name}`,
                                            },
                                        })),
                                    },
                                },
                            },
                        },
                        {
                            "@_Play": "",
                            "@_Rec": "",
                            MarkerTrack: {
                                "@_Name": "Marker",
                                "@_Guid": "00 00 00 00 B1 F5 25 5F 70 04 00 00 28 74 D0 4C",
                            },
                            Track: repeatedSequences.map((item) => ({
                                "@_Guid": generateGUID(),
                                "@_Target": `ShowData.DataPools.Default.Sequences.${item.name}`,
                                "@_Play": "",
                                "@_Rec": "",
                                TimeRange: {
                                    "@_Guid": generateGUID(),
                                    "@_Play": "",
                                    "@_Rec": "",
                                    CmdSubTrack: {
                                        CmdEvent: item.timestamps.map((timestamp, index) => ({
                                            "@_Name": "Go+",
                                            "@_Time": timestamp,
                                            RealtimeCmd: {
                                                "@_Type": "Key",
                                                "@_Source": "Original",
                                                "@_UserProfile": "0",
                                                "@_Status": "On",
                                                "@_IsRealtime": "1",
                                                "@_IsXFade": "0",
                                                "@_IgnoreFollow": "0",
                                                "@_IgnoreCommand": "0",
                                                "@_Assert": "0",
                                                "@_IgnoreNetwork": "0",
                                                "@_FromTriggerNode": "0",
                                                "@_IgnoreExecTime": "0",
                                                "@_IssuedByTimecode": "0",
                                                "@_FromLocalHardwareFader": "1",
                                                ...(index == 0
                                                    ? {
                                                          "@_Object": `ShowData.DataPools.Default.Sequences.${item.name}`,
                                                      }
                                                    : {}),
                                                "@_ExecToken": "Go+",
                                                "@_ValCueDestination": `ShowData.DataPools.Default.Sequences.${item.name}.Cue 1`,
                                            },
                                        })),
                                    },
                                },
                            })),
                        },
                    ],
                },
            },
        };

        return builder.build(obj);
    }
</script>

<svelte:head>
    <title>Reaper to GrandMA3 Converter</title>
    <meta name="description" content="Convert Reaper CSV marker files to GrandMA3 XML format" />
</svelte:head>

<main class="container">
    <header class="header">
        <h1 class="title">🎶 Reaper Markers to GrandMA3 💡</h1>
        <p class="subtitle">Auto-generate your GrandMA3 cues based on Reaper audio markers!</p>
    </header>

    <div class="card">
        <div class="upload-section">
            <label for="file-input" class="upload-label">
                <div bind:this={uploadArea} class="upload-area" class:has-file={fileInput?.files?.length} class:drag-over={isDragOver} class:processing={isProcessing} on:dragover={handleDragOver} on:dragleave={handleDragLeave} on:drop={handleDrop} role="button" tabindex="0" aria-label="Upload CSV file">
                    {#if isProcessing}
                        <div class="spinner"></div>
                        <span class="upload-text processing-text">{processingStatus}</span>
                    {:else}
                        <svg class="upload-icon" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                            <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                            <polyline points="14,2 14,8 20,8"></polyline>
                            <line x1="16" y1="13" x2="8" y2="13"></line>
                            <line x1="16" y1="17" x2="8" y2="17"></line>
                            <polyline points="10,9 9,9 8,9"></polyline>
                        </svg>
                        <span class="upload-text">
                            {#if fileInput?.files?.length}
                                ✅ {fileInput.files[0].name}
                            {:else if isDragOver}
                                📁 Drop your CSV file here
                            {:else}
                                📂 Click to select CSV file or drag & drop
                            {/if}
                        </span>
                        <span class="upload-hint">Supports .csv files from Reaper</span>
                    {/if}
                </div>
            </label>
            <input id="file-input" type="file" accept=".csv" name={Math.random().toString()} bind:this={fileInput} on:change={handleFileChange} class="file-input" />
        </div>

        {#if processingCompleted && fileInput?.files?.length}
            <div class="new-file-section">
                <button class="new-file-button" on:click={clearFile}>
                    <svg class="button-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                        <path d="M3 6h18"></path>
                        <path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"></path>
                        <path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"></path>
                        <line x1="10" y1="11" x2="10" y2="17"></line>
                        <line x1="14" y1="11" x2="14" y2="17"></line>
                    </svg>
                    Clear File - Process New CSV
                </button>
            </div>
        {/if}

        <div class="settings-grid">
            <div class="input-group">
                <label for="sequence-number" class="label">
                    <span class="label-text">Sequence Number</span>
                    <span class="label-hint">Target sequence (1-9999)</span>
                </label>
                <input id="sequence-number" type="number" min="1" max="9999" step="1" bind:value={sequenceNumber} class="input" />
            </div>

            <div class="input-group">
                <label for="prefix" class="label">
                    <span class="label-text">Prefix</span>
                    <span class="label-hint">e.g. Song Number</span>
                </label>
                <input id="prefix" type="text" bind:value={prefix} class="input" />
            </div>

            <div class="input-group" class:disabled={exportMode === "cues-only"}>
                <label for="drive-number" class="label">
                    <span class="label-text">Drive Number</span>
                    <span class="label-hint">Drive to use for import (1-8)</span>
                </label>
                <input id="drive-number" type="number" min="1" max="8" step="1" bind:value={driveNumber} class="input" disabled={exportMode === "cues-only"} />
            </div>
        </div>

        <div class="export-mode-section">
            <div class="label">
                <span class="label-text">Export Mode</span>
            </div>
            <div class="radio-group">
                <label class="radio-label">
                    <input type="radio" bind:group={exportMode} value="cues-and-timecode" class="radio-input" />
                    <span class="radio-text">Cues & Timecode</span>
                </label>
                <label class="radio-label">
                    <input type="radio" bind:group={exportMode} value="cues-only" class="radio-input" />
                    <span class="radio-text">Cues Only</span>
                </label>
            </div>
        </div>
    </div>

    <div class="info-card">
        <h3>How it works:</h3>
        <div class="steps">
            <div class="step">
                <div class="step-number">1</div>
                <div>In Reaper, create a marker for each cue. Use the default color for cues in the master cuestack. Use different colors for each effect sequence (e.g. bass drum, snair, crash, ...) which will become a single sequence with one cue</div>
            </div>
            <div class="step">
                <div class="step-number">2</div>
                <div>Ensure that your time unit in <code>View > Time Unit for Ruler</code> is set to <code>Seconds</code></div>
            </div>
            <div class="step">
                <div class="step-number">3</div>
                <div>Export markers from Reaper as CSV using <code>Actions > Show Actions List > Export Markers (double click)</code></div>
            </div>
            <div class="step">
                <div class="step-number">4</div>
                <div>Upload the CSV file above</div>
            </div>
            <div class="step">
                <div class="step-number">5</div>
                <div>Two XML files will be downloaded automatically</div>
            </div>
            <div class="step">
                <div class="step-number">6</div>
                <div>
                    Move them to the following location:
                    <ul>
                        <li><code>Macro XML:</code> <code>/Users/MaxMustermann/MALightingTechnology/gma3_library/datapools/macros</code></li>
                        <li><code>Timecode XML:</code> <code>/Users/MaxMustermann/MALightingTechnology/gma3_library/datapools/timecodes</code></li>
                    </ul>
                </div>
            </div>
            <div class="step">
                <div class="step-number">7</div>
                <div>In GrandMA3, press Edit, then on an empty slot in the Macros Data Pool and import the Macro XML file</div>
            </div>
            <div class="step">
                <div class="step-number">8</div>
                <div>Execute your macro and have fun!</div>
            </div>
        </div>
    </div>

    <footer class="footer">
        <p>Made with ❤️ and Svelte - View <a href="https://github.com/hrueger/reaper2ma">Source on GitHub</a></p>
        <p>&copy; 2025 Hannes Rüger</p>
    </footer>
</main>

<style>
    :global(:root) {
        /* Light theme variables */
        --bg-gradient-start: #667eea;
        --bg-gradient-end: #764ba2;
        --text-primary: #333;
        --text-secondary: #666;
        --text-white: white;
        --card-bg: white;
        --card-border: rgba(255, 255, 255, 0.2);
        --upload-bg: #fafafa;
        --upload-hover: #f0f4ff;
        --upload-success: #f0fdf4;
        --upload-processing: #fffbeb;
        --border-light: #e5e7eb;
        --border-hover: #9ca3af;
        --accent-blue: #667eea;
        --accent-green: #22c55e;
        --accent-orange: #f59e0b;
        --accent-red: #ef4444;
        --accent-red-dark: #dc2626;
        --input-bg: white;
        --shadow-light: rgba(0, 0, 0, 0.1);
        --shadow-medium: rgba(0, 0, 0, 0.2);
        --info-bg: rgba(255, 255, 255, 0.95);
        --info-border: rgba(255, 255, 255, 0.3);
        --step-bg: #f8fafc;
        --note-bg: #f0f9ff;
        --note-border: #0ea5e9;
    }

    :global(:root) {
        color-scheme: light dark;
    }

    @media (prefers-color-scheme: dark) {
        :global(:root) {
            /* Dark theme variables */
            --bg-gradient-start: #1e1b4b;
            --bg-gradient-end: #312e81;
            --text-primary: #e5e7eb;
            --text-secondary: #9ca3af;
            --text-white: #f9fafb;
            --card-bg: #1f2937;
            --card-border: rgba(75, 85, 99, 0.3);
            --upload-bg: #374151;
            --upload-hover: #1e3a8a;
            --upload-success: #064e3b;
            --upload-processing: #451a03;
            --border-light: #4b5563;
            --border-hover: #6b7280;
            --accent-blue: #818cf8;
            --accent-green: #34d399;
            --accent-orange: #fbbf24;
            --accent-red: #f87171;
            --accent-red-dark: #ef4444;
            --input-bg: #374151;
            --shadow-light: rgba(0, 0, 0, 0.3);
            --shadow-medium: rgba(0, 0, 0, 0.5);
            --info-bg: rgba(31, 41, 55, 0.95);
            --info-border: rgba(75, 85, 99, 0.3);
            --step-bg: #4b5563;
            --note-bg: #1e3a8a;
            --note-border: #3b82f6;
        }
    }

    :global(body) {
        margin: 0;
        padding: 0;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
        background: linear-gradient(135deg, var(--bg-gradient-start) 0%, var(--bg-gradient-end) 100%);
        min-height: 100vh;
        color: var(--text-primary);
        transition:
            background 0.3s ease,
            color 0.3s ease;
    }

    .container {
        max-width: 800px;
        margin: 0 auto;
        padding: 2rem 1rem;
        min-height: 100vh;
        display: flex;
        flex-direction: column;
        gap: 2rem;
    }

    .header {
        text-align: center;
        color: var(--text-white);
        margin-bottom: 1rem;
    }

    .title {
        font-size: 2.5rem;
        font-weight: 700;
        margin: 0 0 0.5rem 0;
        text-shadow: 0 2px 4px var(--shadow-light);
    }

    .subtitle {
        font-size: 1.1rem;
        opacity: 0.9;
        margin: 0;
        font-weight: 300;
    }

    .card {
        background: var(--card-bg);
        border-radius: 16px;
        padding: 2rem;
        box-shadow: 0 10px 30px var(--shadow-medium);
        backdrop-filter: blur(10px);
        border: 1px solid var(--card-border);
        transition:
            background 0.3s ease,
            border 0.3s ease;
    }

    .upload-section {
        margin-bottom: 2rem;
    }

    .upload-label {
        display: block;
        cursor: pointer;
        transition: transform 0.2s ease;
    }

    .upload-label:hover {
        transform: translateY(-2px);
    }

    .upload-area {
        border: 3px dashed var(--border-light);
        border-radius: 12px;
        padding: 3rem 2rem;
        text-align: center;
        background: var(--upload-bg);
        transition: all 0.3s ease;
        position: relative;
        overflow: hidden;
    }

    .upload-area:hover {
        border-color: var(--accent-blue);
        background: var(--upload-hover);
    }

    .upload-area.has-file {
        border-color: var(--accent-green);
        background: var(--upload-success);
        border-style: solid;
    }

    .upload-area.drag-over {
        border-color: var(--accent-blue);
        background: var(--upload-hover);
        transform: scale(1.02);
        box-shadow: 0 0 20px rgba(102, 126, 234, 0.3);
    }

    .upload-area.processing {
        border-color: var(--accent-orange);
        background: var(--upload-processing);
        pointer-events: none;
    }

    .upload-icon {
        color: var(--text-secondary);
        margin-bottom: 1rem;
        transition: color 0.3s ease;
    }

    .upload-area:hover .upload-icon {
        color: var(--accent-blue);
    }

    .upload-area.has-file .upload-icon {
        color: var(--accent-green);
    }

    .upload-text {
        display: block;
        font-size: 1.1rem;
        font-weight: 500;
        margin-bottom: 0.5rem;
        color: var(--text-primary);
    }

    .upload-hint {
        font-size: 0.9rem;
        color: var(--text-secondary);
        display: block;
    }

    .processing-text {
        color: var(--accent-orange) !important;
        font-weight: 600;
    }

    .spinner {
        width: 32px;
        height: 32px;
        border: 3px solid var(--border-light);
        border-top: 3px solid var(--accent-orange);
        border-radius: 50%;
        animation: spin 1s linear infinite;
        margin-bottom: 1rem;
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    .file-input {
        position: absolute;
        opacity: 0;
        pointer-events: none;
    }

    .new-file-section {
        margin: 1.5rem 0;
        text-align: center;
    }

    .new-file-button {
        display: inline-flex;
        align-items: center;
        gap: 0.5rem;
        padding: 0.75rem 1.5rem;
        background: linear-gradient(135deg, var(--accent-red), var(--accent-red-dark));
        color: var(--text-white);
        border: none;
        border-radius: 8px;
        font-size: 0.9rem;
        font-weight: 500;
        cursor: pointer;
        transition: all 0.2s ease;
        box-shadow: 0 2px 4px var(--shadow-light);
    }

    .new-file-button:hover {
        background: linear-gradient(135deg, var(--accent-red-dark), #b91c1c);
        transform: translateY(-1px);
        box-shadow: 0 4px 8px var(--shadow-medium);
    }

    .new-file-button:active {
        transform: translateY(0);
    }

    .button-icon {
        flex-shrink: 0;
    }

    .export-mode-section {
        margin-top: 1.5rem;
    }

    .radio-group {
        display: grid;
        grid-template-columns: repeat(2, 1fr);
        gap: 1rem;
        margin-top: 0.75rem;
    }

    .radio-label {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        padding: 0.75rem 1rem;
        border: 2px solid var(--border-light);
        border-radius: 8px;
        cursor: pointer;
        transition: all 0.2s ease;
        background: var(--upload-bg);
    }

    .radio-label:hover {
        border-color: var(--accent-blue);
        background: var(--upload-hover);
    }

    .radio-input {
        width: 18px;
        height: 18px;
        cursor: pointer;
        accent-color: var(--accent-blue);
    }

    .radio-text {
        color: var(--text-primary);
        font-size: 0.95rem;
        font-weight: 500;
    }

    .radio-label:has(.radio-input:checked) {
        border-color: var(--accent-blue);
        background: var(--upload-hover);
        box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
    }

    .settings-grid {
        display: grid;
        grid-template-columns: repeat(3, minmax(0, 1fr));
        gap: 2rem;
    }

    .input-group {
        display: flex;
        flex-direction: column;
    }

    .input-group.disabled {
        opacity: 0.5;
        pointer-events: none;
    }

    .label {
        margin-bottom: 0.5rem;
        display: block;
    }

    .label-text {
        font-weight: 600;
        color: var(--text-primary);
        font-size: 0.95rem;
        display: block;
        margin-bottom: 0.25rem;
    }

    .label-hint {
        font-size: 0.8rem;
        color: var(--text-secondary);
        display: block;
        font-weight: normal;
    }

    .input {
        padding: 0.75rem 1rem;
        border: 2px solid var(--border-light);
        border-radius: 8px;
        font-size: 1rem;
        transition: all 0.2s ease;
        background: var(--input-bg);
        color: var(--text-primary);
    }

    .input:focus {
        outline: none;
        border-color: var(--accent-blue);
        box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
    }

    .input:hover {
        border-color: var(--border-hover);
    }

    .info-card {
        background: var(--info-bg);
        border-radius: 16px;
        padding: 2rem;
        backdrop-filter: blur(10px);
        border: 1px solid var(--info-border);
        color: var(--text-primary);
        transition:
            background 0.3s ease,
            border 0.3s ease;
    }

    .info-card h3 {
        margin: 0 0 1.5rem 0;
        color: var(--text-primary);
        font-size: 1.4rem;
        text-align: center;
    }

    .steps {
        display: grid;
        gap: 1rem;
        margin-bottom: 1.5rem;
    }

    .step {
        display: flex;
        align-items: center;
        gap: 1rem;
        padding: 1rem;
        background: var(--step-bg);
        border-radius: 8px;
        border-left: 4px solid var(--accent-blue);
        transition: background 0.3s ease;
    }

    .step-number {
        background: var(--accent-blue);
        color: var(--text-white);
        width: 32px;
        height: 32px;
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        font-weight: 600;
        flex-shrink: 0;
    }

    .note {
        background: var(--note-bg);
        padding: 1rem;
        border-radius: 8px;
        border-left: 4px solid var(--note-border);
        margin: 0;
        font-size: 0.9rem;
        line-height: 1.5;
        color: var(--text-primary);
        transition:
            background 0.3s ease,
            border 0.3s ease;
    }

    .footer {
        text-align: center;
        color: rgba(255, 255, 255, 0.8);
        font-size: 0.9rem;
        margin-top: auto;
        padding-top: 1rem;
    }

    @media (max-width: 640px) {
        .container {
            padding: 1rem;
        }

        .title {
            font-size: 2rem;
        }

        .card {
            padding: 1.5rem;
        }

        .radio-group {
            grid-template-columns: 1fr;
        }

        .settings-grid {
            grid-template-columns: 1fr;
            gap: 1.5rem;
        }

        .upload-area {
            padding: 2rem 1rem;
        }
    }

    /* Success animation */
    @keyframes success {
        0% {
            transform: scale(1);
        }
        50% {
            transform: scale(1.05);
        }
        100% {
            transform: scale(1);
        }
    }

    .upload-area.has-file {
        animation: success 0.6s ease-out;
    }

    /* Focus styles for accessibility */
    .upload-label:focus-within .upload-area {
        outline: 2px solid #667eea;
        outline-offset: 2px;
    }
</style>
