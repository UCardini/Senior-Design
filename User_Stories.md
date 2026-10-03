# Assignment 4 (User Stories and Use cases)

**Project** Steganography done on JPEG vs. PNG (does image compression matter)

**Team Members**: Carter Smith, Alex Watts  

## Stakeholder Map

Primary: Digital forensics analyst

Secondary: Steganography researcher

Hidden: Security software developer

## User Stories

US-01 (primary): As a digital forensics analyst, I want to know if hidden messages are harder to detect in JPEG images than within PNG images, so that I know which images are vulnerable.

US-02 (secondary): As a image security researcher, I want to see detection results for JPEG and PNG images side by side, so that I can decide which format to use in my demonstration.

US-03 (hidden): As a security software developer, I want to know which file type hides information better, so that I can make systems check those files more closely.

## Invest Self-Check

### US-01
* Independent: PASS
* Negotiable: PASS
* Valuable: PASS
* Estimable: PASS
* Small: PASS
* Testable: PASS

### US-02
* Independent: PASS
* Negotiable: PASS
* Valuable: PASS
* Estimable: PASS
* Small: PASS
* Testable: PASS

### US-03
* Independent: PASS
* Negotiable: PASS
* Valuable: PASS
* Estimable: PASS
* Small: PASS
* Testable: PASS

## Use Cases

### UC-01: Compare detection for JPEG and PNG files
* Expands: US-01
* Primary actor: Digital forensics analyst
* Secondary actor: Detection tool
* Preconditions:
    * Detection tool is installed
    * 50 JPEG and 50 PNG images each have a hidden message
* Main flow:
    1. Analyst loads the images into the tool
    2. Tool scans each image
    3. Analyst asks for the results
    4. Tool shows how many hidden messages it found in each format
* Alternate flow:
    * Analyst loads only one format
    * Tool shows results for that format only
* Exception flow:
    * Tool can't open an image
    * Tool skips it and lists it as an error
* Postcondition:
    * Number of hidden messages found in each format is recorded

## Acceptance Criteria

### AC-01.1 (Main flow)
* Given: 50 JPEG and 50 PNG images, each with a hidden message
* When: The analyst scans all 100 images
* Then: The tool shows the number of hidden messages found in each format

### AC-01.2 (Exception flow)
* Given: 1 image the tool can't open
* When: The analyst scans the images
* Then: The tool skips that image and lists its file name as an error
