# [cite_start]🔬 SOP: Automated Algae Counting (ImageJ) [cite: 1]

| Field | Detail |
| :--- | :--- |
| **Date** | [cite_start]November 21, 2025 [cite: 2] |
| **Scope** | [cite_start]Fluorescence Microscopy & Hemocytometer Analysis [cite: 3] |

---

## 1. Purpose

[cite_start]To automate the counting of algal cells (e.g., *Symbiodiniaceae*, Phytoplankton) from fluorescence microscope images[cite: 5].

This protocol uses a custom ImageJ macro to:
* [cite_start]**Standardize the Count Area:** Automatically crops the image to the exact volume of the Hemocytometer Central Square (0.1 µL)[cite: 7].
* [cite_start]**Remove Bias:** Uses mathematical thresholding ("Triangle") rather than human eye estimation[cite: 8].
* [cite_start]**Handle Variable Replicates:** Automatically adjusts to any number of photos per sample[cite: 9].

---

## 2. Phase 0: Imaging Protocol (Critical)

[cite_start]To ensure the automated script calculates volume correctly, all images **must** be taken using these exact settings[cite: 11].

| Setting | Requirement |
| :--- | :--- |
| **Magnification** | [cite_start]10X Objective (Total Mag: 100X)[cite: 12]. |
| **Warning** | [cite_start]Do not use 20X or 40X for counting[cite: 13]. |
| **Fluorescence Channel** | [cite_start]Use the **Chlorophyll / Red** channel (Excitation: ~450-480nm, Emission: >600nm)[cite: 20]. |
| **Exposure** | [cite_start]Adjust so cells are bright but **not** saturated[cite: 21]. [cite_start]Background should be dark[cite: 21]. |

### Camera Settings (Binning)
* [cite_start]**Hemocytometer (Brightfield):** Full Resolution (Binning 1x1)[cite: 15]. [cite_start]Expected Size: ~1600 x 1600 pixels[cite: 16].
* [cite_start]**Algae (Fluorescence):** High Sensitivity (Binning 3x3)[cite: 17]. [cite_start]Expected Size: ~536 x 536 pixels[cite: 18].
    > [cite_start]**Note:** If you change the binning, you must update the `boxSize` in the script[cite: 19].

---

## [cite_start]3. The Logic: Why It Works [cite: 22]

[cite_start]Before running the script, understanding the core parameters is critical[cite: 23].

| Parameter | Setting/Value | Rationale |
| :--- | :--- | :--- |
| **The "Cut" (Volume)** | [cite_start]512x512 box [cite: 28] | [cite_start]The script creates a 512x512 box[cite: 28]. [cite_start]Anything inside represents exactly **0.1 µL** of sample[cite: 28]. (Derived from Standard Neubauer Central Square = 1mm x 1mm, Calibrated Brightfield Width = 1535 px, Fluorescence Width / 3 = 512 px) [cite_start][cite: 25, 26, 27]. |
| **The Threshold (Detection)** | [cite_start]**"Triangle"** Method [cite: 30] | [cite_start]This is optimized for "dim" cells and ignores background static better than the standard "Otsu" method[cite: 30]. |
| **The Size Filter (Noise)** | [cite_start]**> 50 pixels** [cite: 32] | [cite_start]This removes dust (typically ~14px) while keeping small algae (~240px)[cite: 32]. |

---

## [cite_start]4. Procedure [cite: 33]

### [cite_start]Phase 1: Preparation [cite: 34]
1.  [cite_start]**Organize Files:** Place all your .tif, .jpg, or .png images into a single **Input Folder**[cite: 35].
2.  **Naming Convention:** SampleID\_Replicate.tif (e.g., `401_1.tif`, `401_2.tif`). [cite_start]The script groups replicates automatically[cite: 36].
3.  [cite_start]**Create Output Folder:** Create an empty folder named "Results" or "Counted\_Images"[cite: 37].

### [cite_start]Phase 2: Running the Script [cite: 38]
1.  [cite_start]Open **Fiji / ImageJ**[cite: 39].
2.  [cite_start]Drag the script file (`AlgaeCounter.ijm`) into the Fiji bar (or go to `Plugins` > `Macros` > `Run`)[cite: 40].
3.  [cite_start]**Select Input Folder:** Choose the folder with your images[cite: 41].
4.  [cite_start]**Select Output Folder:** Choose your results folder[cite: 42].

### [cite_start]Phase 3: The "Eraser" Step (Interactive) [cite: 43]
[cite_start]The script will open images one by one and pause[cite: 44, 45]. For **EACH** image:
1.  [cite_start]Look for the **Scale Bar** (or any large debris clumps)[cite: 46].
2.  [cite_start]Use the mouse to **Draw a Box** over the text/scale bar[cite: 47].
3.  [cite_start]Click **OK** on the prompt window[cite: 48].
4.  [cite_start]The script will paint the box black (ignoring it), count the algae, and move to the next image[cite: 49].

### [cite_start]Phase 4: Results & Quality Control [cite: 50]

#### [cite_start]Extracting Data to Excel [cite: 51]
1.  [cite_start]When finished, a window named **"Log"** will appear[cite: 52].
2.  [cite_start]Click inside the **"Log"** window[cite: 53].
3.  [cite_start]Press `Ctrl + A` (Select All) then `Ctrl + C` (Copy)[cite: 54].
4.  [cite_start]Paste into Excel[cite: 55].
5.  [cite_start]**Column Layout:** Sample Name -> Concentration -> Average -> Count 1 -> Count 2...[cite: 56].
    > [cite_start]**Note:** Column B is always the Final Result[cite: 57].

#### [cite_start]Visual Verification [cite: 58]
1.  [cite_start]Open the output folder and check images named `Checked_....`[cite: 59].
2.  [cite_start]**Cyan Numbers** indicate valid cells[cite: 60].
3.  [cite_start]Verify that the scale bar is gone and faint cells are numbered[cite: 61].

---

## [cite_start]5. The Script Code [cite: 62]

Copy the code below and save it as **`AlgaeCounter.ijm`**. [cite_start]To install in ImageJ: `Plugins` > `New` > `Macro`, paste code, and `Save`[cite: 63].

```ijm
/*
* SOP SCRIPT: AUTOMATED ALGAE COUNTER (Variable Replicates)
* Lab Protocol: 11/2025
* * PARAMETERS:
* - Crop Size: 512x512 (Calculated for 10X Binning 3)
* - Min Size: 50 pixels (Filters dust <14px)
* - Threshold: Triangle (Optimized for dim fluorescence)
*/
macro "SOP Algae Variable [F2]" {
    // --- CONFIGURATION ---
    minSize = 50;          
    boxSize = 512; 
    threshMethod = "Triangle"; 
    
    // --- SETUP ---
    run("Close All");
    print("\Clear");
    roiManager("Reset");
    
    // Folder Selection
    inputDir = getDirectory("Select the INPUT Folder (Images)");
    outputDir = getDirectory("Select the OUTPUT Folder (Results)");
    
    list = getFileList(inputDir);
    Array.sort(list); 
    
    // Excel Header
    print("Sample No\tConc (Cells/mL)\tAverage\tCount 1\tCount 2\tCount 3\t(etc...)");

    currentSample = "";
    counts = newArray(0); 
    
    // Interactive Mode (BatchMode OFF to allow erasing)
    setBatchMode(false); 

    for (i = 0; i < list.length; i++) {
        filename = list[i];
        if (endsWith(filename, ".tif") || endsWith(filename, ".jpg") || endsWith(filename, ".png")) {
            
            // 1. Parse Sample ID
            dotIndex = lastIndexOf(filename, ".");
            nameNoExt = substring(filename, 0, dotIndex);
            parts = split(nameNoExt, "_");
            sampleID = parts[0]; 

            // 2. Data Grouping
            if (sampleID != currentSample) {
                if (currentSample != "") { printRow(currentSample, counts); }
                currentSample = sampleID;
                counts = newArray(0); 
            }
            
            // 3. Image Prep
            open(inputDir + filename);
            originalTitle = getTitle();
            
            // Auto-Crop to 0.1 uL Volume
            makeRectangle(getWidth()/2 - boxSize/2, getHeight()/2 - boxSize/2, boxSize, boxSize);
            run("Crop");
            
            // 4. USER INTERACTION: Scale Bar Eraser
            setTool("rectangle");
            waitForUser("SOP Action", "Draw a box over the SCALE BAR text.\nThen click OK.");
            
            if (selectionType() != -1) {
                run("Set...", "value=0"); // Paint black
                run("Select None");
            }
            // 5. Analysis
            run("Duplicate...", "title=Detection_Mask");
            run("8-bit");
            run("Subtract Background...", "rolling=50");
            setAutoThreshold(threshMethod + " dark");
            run("Convert to Mask");
            run("Watershed");
            
            // Count
            roiManager("Reset");
            run("Analyze Particles...", "size=" + minSize + "-Infinity circularity=0.30-1.00 exclude add");
            
            thisCount = roiManager("count");
            counts = Array.concat(counts, thisCount);
            
            // 6. Save Evidence
            selectWindow(originalTitle);
            run("8-bit");
            roiManager("Show All with labels");
            run("Labels...", "color=Cyan font=14 show use draw");
            run("Flatten"); 
            saveAs("Jpeg", outputDir + "Checked_" + filename);
            
            close("*"); 
            roiManager("Reset");
        }
    }

    // Print Final Row
    if (currentSample != "") { printRow(currentSample, counts); }
    
    showMessage("SOP Complete", "All images processed.\n\n1. Click inside the LOG window.\n2. Press Ctrl+A (Select All).\n3. Press Ctrl+C (Copy).\n4. Paste into Excel.");
}

function printRow(sample, cArr) {
    sum = 0; n = cArr.length;
    
    // Calculate Stats
    for (k=0; k<n; k++) { sum = sum + cArr[k]; }
    avg = 0; if (n > 0) { avg = sum / n; }
    conc = avg * 10000; 
    
    // Stats First Layout
    line = sample + "\t" + conc + "\t" + avg;
    
    // Append Variable Counts
    for (k=0; k<n; k++) {
        line = line + "\t" + cArr[k];
    }
    print(line);
}