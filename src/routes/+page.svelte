<script lang="ts">
    import * as csv from "@vanillaes/csv";
    import { XMLBuilder } from "fast-xml-parser";

    let fileInput: HTMLInputElement;
    let sequenceNumber = 101;
    let driveNumber = 2;

    type ParsedData = {
        name: string;
        start: string;
    }[];
    type FullData = {
        data: ParsedData;
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

    function processCsv(dataString: string, filename: string) {
        const parsedLines = csv.parse(dataString) as string[][];
        const header = parsedLines[0];
        const dataByHeader = parsedLines.slice(1).map((row) => {
            const obj: Record<string, string> = {};
            row.forEach((value, index) => {
                obj[header[index]] = value;
            });
            return obj;
        }) as { "#": string; Name: string; Start: string }[];

        const data = dataByHeader.map((item) => ({
            name: item.Name.replace(/[^a-zA-Z0-9äöüÄÖÜß \-_#%\/\(\)\[\]=+]/g, ""),
            // name: item.Name.replace(/[^a-z0-9]/g, ""),
            start: item.Start,
        }));

        const xmlContent = generateMacroXML({ data, filename });
        downloadContentWithName(xmlContent, `${filename}_macro.xml`);
        const timecodeContent = generateTimecodeXML({ data, filename });
        downloadContentWithName(timecodeContent, `${filename}_timecode.xml`);
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

    function generateMacroXML({ data, filename }: FullData): string {
        const obj = {
            ...XML_HEADER,
            GMA3: {
                "@_DataVersion": "1.4.0.2",
                Macro: {
                    "@_Name": `Macro ${filename}`,
                    "@_Guid": "00 00 00 00 A8 F8 B9 20 78 06 00 00 A5 46 09 AA",
                    MacroLine: [
                        {
                            "@_Command": `Store Sequence ${sequenceNumber} Cue 1 thru ${data.length}`,
                            "@_Wait": "0.10",
                        },
                        ...data.map((item, index) => ({
                            "@_Command": `Label Sequence ${sequenceNumber} Cue ${index + 1} "${item.name}"`,
                            "@_Wait": "0.10",
                        })),
                        {
                            "@_Command": `Drive ${driveNumber}`,
                            "@_Wait": "0.10",
                        },
                        {
                            "@_Command": `import Timecode "${filename}_timecode"`,
                            "@_Wait": "0.10",
                        },
                    ],
                },
            },
        };

        return builder.build(obj);
    }

    function generateTimecodeXML({ data, filename }: FullData): string {
        const obj = {
            ...XML_HEADER,
            GMA3: {
                "@_DataVersion": "1.4.0.2",
                Timecode: {
                    "@_Name": filename,
                    "@_Guid": "00 00 00 00 3F 76 B7 04 32 0B 00 00 68 A1 4F AA",
                    "@_Cursor": "00.00",
                    "@_Duration": data.length > 0 ? (parseFloat(data[data.length - 1].start) + 1).toFixed(3) : "0.00",
                    "@_LoopCount": "0",
                    "@_TCSlot": "-1",
                    "@_SwitchOff": "Keep Playbacks",
                    "@_Goto": "as Go",
                    "@_Timedisplayformat": "<10d11h23m45>",
                    "@_FrameReadout": "<Seconds>",
                    TrackGroup: {
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
                                    CmdEvent: data.map((item, index) => ({
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
                },
            },
        };

        return builder.build(obj);
    }
</script>

<input type="file" bind:this={fileInput} on:change={handleFileChange} />
<br />
Sequence Number: <input type="number" min="1" max="9999" step="1" bind:value={sequenceNumber} />
<br />
Drive Number: <input type="number" min="1" max="8" step="1" bind:value={driveNumber} />
