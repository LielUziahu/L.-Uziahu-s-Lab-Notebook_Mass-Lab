# 🔬 SOP: Automated Algae Counting (ImageJ)

| Field | Detail |
| :--- | :--- |
| **Date** | November 21, 2025 |
| **Scope** | Fluorescence Microscopy & Hemocytometer Analysis |

---

## 1. Purpose

To automate the counting of algal cells (e.g., *Symbiodiniaceae*, Phytoplankton) from fluorescence microscope images.

This protocol uses a custom ImageJ macro to:
* **Standardize the Count Area:** Automatically crops the image to the exact volume of the Hemocytometer Central Square (0.1 µL).
* **Remove Bias:** Uses mathematical thresholding ("Triangle") rather than human eye estimation.
* **Handle Variable Replicates:** Automatically adjusts to any number of photos per sample.

---

## 2. Phase 0: Imaging Protocol (Critical)

To ensure the automated script calculates volume correctly, all images **must** be taken using these exact settings.

| Setting | Requirement |
| :--- | :--- |
| **Magnification** | 10X Objective (Total Mag: 100X). Do not use 20X or 40X for counting. |
| **Fluorescence Channel** | Use the **Chlorophyll / Red** channel (Excitation: ~450-480nm, Emission: >600nm). |
| **Exposure** | Adjust so cells are bright but **not** saturated. Background should be dark. |

### Camera Settings (Binning)
* **Hemocytometer (Brightfield):** Full Resolution (Binning 1x1). Expected Size: ~1600 x 1600 pixels.
* **Algae (Fluorescence):** High Sensitivity (Binning 3x3). Expected Size: ~536 x 536 pixels.
    > **Note:** If you change the binning, you must update the `boxSize` in the script.

---

## 3. The Logic: Why It Works

Before running the script, understanding the core parameters is critical.

| Parameter | Setting/Value | Rationale |
| :--- | :--- | :--- |
| **The "Cut" (Volume)** | 512x512 box | Anything inside represents exactly **0.1 µL** of sample. (Derived from Standard Neubauer Central Square = 1mm x 1mm, Calibrated Brightfield Width = 1535 px, Fluorescence Width / 3 = 512 px). |
| **The Threshold (Detection)** | **"Triangle"** Method | This is optimized for "dim" cells and ignores background static better than the standard "Otsu" method. |
| **The Size Filter (Noise)** | **> 50 pixels** | This removes dust (typically ~14px) while keeping small algae (~240px). |

---

## 4. Procedure

### Phase 1: Preparation
1.  **Organize Files:** Place all your .tif, .jpg, or .png images into a single **Input Folder**.
2.  **Naming Convention:** SampleID\_Replicate.tif (e.g., `401_1.tif`, `401_2.tif`). The script groups replicates automatically.
3.  **Create Output Folder:** Create an empty folder named "Results" or "Counted\_Images".

### Phase 2: Running the Script
1.  Open **Fiji / ImageJ**.
2.  Drag the script file (`AlgaeCounter.ijm`) into the Fiji bar (or go to `Plugins` > `Macros` > `Run`).
3.  **Select Input Folder:** Choose the folder with your images.
4.  **Select Output Folder:** Choose your results folder.

### Phase 3: The "Eraser" Step (Interactive)
The script will open images one by one and pause. For **EACH** image:
1.  Look for the **Scale Bar** (or any large debris clumps).
2.  Use the mouse to **Draw a Box** over the text/scale bar.
3.  Click **OK** on the prompt window.
4.  The script will paint the box black (ignoring it), count the algae, and move to the next image.

### Phase 4: Results & Quality Control

#### Extracting Data to Excel
1.  When finished, a window named **"Log"** will appear.
2.  Click inside the **"Log"** window.
3.  Press `Ctrl + A` (Select All) then `Ctrl + C` (Copy).
4.  Paste into Excel.
5.  **Column Layout:** Sample Name -> Concentration -> Average -> Count 1 -> Count 2....
    > **Note:** Column B is always the Final Result.

#### Visual Verification
1.  Open the output folder and check images named `Checked_....`.
2.  **Cyan Numbers** indicate valid cells.
3.  Verify that the scale bar is gone and faint cells are numbered.

---

## 5. The Script Code

Copy the code below and save it as **`AlgaeCounter.ijm`**. To install in ImageJ: `Plugins` > `New` > `Macro`, paste code, and `Save`.

AlgaeCounter.ijm
{
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
    print("\\Clear");
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
}
