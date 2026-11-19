# User Stories and Use Cases
**Project:** JPEG vs. PNG Steganography
**Team:** Carter Smith, Alex Watts

---

## Stakeholder Map

| Type | Stakeholder |
|------|-------------|
| Primary | Digital forensics analyst |
| Secondary | Image security researcher |
| Hidden | Security software developer |

---

## User Stories

**US-01 (primary):** As a digital forensics analyst, I want to know if hidden messages are harder to detect in JPEG images than within PNG images, so that I know which images are vulnerable.

**US-02 (secondary):** As an image security researcher, I want to see detection results for JPEG and PNG images side by side, so that I can decide which format to use in my demonstration.

**US-03 (hidden):** As a security software developer, I want to know which file type hides information better, so that I can make systems check those files more closely.

---

## INVEST Self-Check

**US-01** 

* Independent: `PASS`
* Negotiable: `PASS`
* Valuable: `PASS`
* Estimable: `PASS`
* Small: `PASS`
* Testable: `PASS`

**US-02** 

* Independent: `PASS`
* Negotiable: `PASS`
* Valuable: `PASS`
* Estimable: `PASS`
* Small: `PASS`
* Testable: `PASS`

**US-03** 
* Independent: `PASS`
* Negotiable: `PASS`
* Valuable: `PASS`
* Estimable: `PASS`
* Small: `PASS`
* Testable: `PASS`

---

## Use Cases

### UC-01: Compare detection rates for JPEG and PNG

- **Expands:** US-01
- **Primary actor:** Digital forensics analyst
- **Secondary actor:** Detection tool 
- **Preconditions:**
  - Detection tool is installed 
  - 50 JPEG and 50 PNG images each contain an embedded hidden message
- **Main flow:**
  1. Analyst runs the pipeline script using the image set
  2. Script passes each image to the detection tool
  3. Detection tool returns a score for each image
  4. Script records each score to a CSV file
  5. Analyst reviews the CSV for detection rates per format
- **Alternate flow:**
  - Analyst runs the pipeline against one format only
  - Script produces results for only that format (other gets skipped)
- **Exception flow:**
  - Detection tool cannot open an image 
  - Script logs the filename as an error and continues
- **Postcondition:**
  - Detection scores for all processed images are recorded in the output CSV, with errors logged separately

---

## Acceptance Criteria

**AC-01.1** (Main flow)
- Given: 50 JPEG and 50 PNG cover images plus 50 JPEG and 50 PNG stego images
- When: The analyst runs the full pipeline
- Then: The output CSV contains 200 rows with columns file, format, label, score, and msg_ber, and the script completes without error

**AC-01.2** (Exception flow)
- Given: One image file is corrupt and cannot be opened by the detection tool
- When: The analyst runs the pipeline
- Then: The script skips that image, records its filename in an errors log, and completes processing the remaining images without raising an unhandled exception
