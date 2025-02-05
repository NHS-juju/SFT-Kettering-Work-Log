# SFT-Kettering-Work-Log

<a href="https://www.somersetft.nhs.uk/">
	<img alttext="Somerset NHS Foundation Trust Logo" src="https://www.somersetft.nhs.uk/wp-content/uploads/2023/08/Somerset-NHS-logo_EPS-FILE_right-alligned-logo-01-e1690903130748-1024x517.jpg" width="280" />
</a>

# Somerset NHS Foundation Trust - Migrating from Kettering+HTML to Kettering+PDF

> Work logs 

## Overview

### Purpose

This repo outlines the steps that have been taken to migrate from producing `Kettering+html` to `Kettering+pdf` for sending clinical documents via [MESH](https://digital.nhs.uk/services/message-exchange-for-social-care-and-health-mesh).

### Background

Somerset NHS Foundation Trust has a complex technical estate born from being formed from three Trusts merging over the past few years and in turn hold some technical debt. The Trust generates a vast number of clinical documents for patients under the Trusts care across Somerset. Every encounter with a patient generates a document, usually an inpatient discharge summary, emergency discharge summary, or outpatient clinic letter.

GP Practice Management Systems (PMS) support receiving electronic documents from Secondary Care (and other organisations) using a file type known as `kettering` or `ket` which is an xml file matching a specific schema. These originally supported a html payload, and specification was later expanded to support a `pdf` payload. Unfortunately, this change is not well documented, and information on this is difficult to come by. Additionally, at least one PMS supports the payload being `doc`/`docx` however this not being the case across all PMS systems authorised for use by the NHS makes this an unfavourable option when attempting to work towards a one-size-fits-all solution.

There are IT systems across the Trust that natively produce `pdf`, produce `doc`/`docx` documents, or even produce `html` - all of which are received into an Integration Engine[^1]. Unfortunately, due to the above mentioned knowledge gap regarding PDFs being supported, the starting point for this work started life with one of the following flows depending on the the system in use and the hospital site it is predominantly used in:
`pdf`>`html`>`Kettering+html`
`doc`/`docx`>`html`>`Kettering+html`
`pdf`>`doc`/`docx`>`html`>`Kettering+html`
`html`>`Kettering+html`
[^1]:The received documents are nearly always processed as part of a message made up of `HL7` or `xml`, however some are simply the actual document file and additional work is then done to identify the patient the document relates to either from the filename or content of the document. To avoid over complication, the rest of this document will simply refer to the documents and not their wrapper unless necessary.

This in itself brings a number of issues, mainly
1. Each conversion jump risks the content of the document becoming malformed in some way.
2. There are costs to some software used for the conversions, which add up each year.

Even for instances where the `html` was rendered perfectly or without being converted between different formats, it was not without risks. In years gone by there was a time where there could be major differences in how the `html` would be rendered based on what version of Internet Explorer was being used. There were also cases where receiving systems would [alter the content of the HTML by stripping out various `html` tags (including image tags, removing any images being attached)](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#removed-images-examples) as well as malformed the documents and [removing important content](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#missing-content-examples) or [overlaying paragraphs of text making the content unreadable](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#overlapping-paragraph-examples). Due to these issues + others, it was not uncommon for Trusts within a county to avoid sending documents to other counties in case their PMS was from a vendor they had not worked with and these issues exist. However, in the case of Yeovil District Hospital, this was not possible due to being situated so close to the boarder with Dorset, and the examples included in this section highlights the types of issues seen.

### The NHS App, and a renewed interest in improving the status-quo
In October 2023 the Trust was contacted by the [Somerset ICB](https://nhssomerset.nhs.uk/) to see if we could look at what could be done to alter format of documents sent from the Trust to GP Practices within Somerset as sending them in their current format meant that they were present in the PMS [EMISWeb](https://www.emishealth.com/products/emis-web) as a `ket` file, which was then not able to be displayed within the NHS App for patients to view.

This led to meetings with product managers from [EMISHealth](https://www.emishealth.com/) in November 2023 who confirmed that they were able to receive the payloads within the `ket` files as a BASE64 encoded `pdf` or `doc`/`docx`. Development work was then undertaken by the Trust using the Integration Engine [InterSystems Health Connect](https://www.intersystems.com/uk/products/healthshare/health-connect/).

### Initial Development Work
The basis of the technical work actually came from example code that was shared with the Trust by contacts at InterSystems UK. Part of this code has a message class that, once populated and saved, produces the `XML` portion of the `ket` as well as adding in the payload. What was interesting is that some comments within the code indicates that the ability to have a `pdf` payload was added in sometime in 2019. As this work was being done rather myopically and the revelation that not all PMS's support anything more than `html` or `pdf` payloads had not yet occurred, work was done to extend this codebase to also support `doc`/`docx`.

Once an example output could be produced, this was shared with contacts at [EMISHealth](https://www.emishealth.com/) to test the processing of the generated files from the Integration Engine. This was found to have been successful, so the next steps were to begin transitioning to the new setup. This consisted of updating the existing routes within the Integration Engine [InterSystems Health Connect](https://www.intersystems.com/uk/products/healthshare/health-connect/) to build the messages into a common message with either a `pdf` or `doc`/`docx` payload and then use this to then output the files required for outputting to the GP systems [via MESH](https://digital.nhs.uk/services/message-exchange-for-social-care-and-health-mesh)

### Path-To-Live (Attempt 1)
The plan for moving forward with the works to move from the existing setup was simple:
1. Deploy the change for `system` output into the Live environment
2. Contact a number of GP practices and confirm that they are able to view the documents we have sent
3. Rinse and repeat.

The first few systems attempted with this were outputting `pdf` files, which meant that the tests were successful. However, the first tests with a system outputting `doc`/`docx` failed when checking with a practice using [SystmOne](https://en.wikipedia.org/wiki/SystmOne) and a quick confirmation from their support team confirmed [^2] that they only support Kettering+HTML[^3] or Kettering+PDF . Therefore the change for this system was rolled back.
[^2]: It Is sometimes difficult engaging with suppliers in this context as we're not their customer, so this was achieved by asking the affected practice to contact the supplier on our behalf. Thankfully, the response was very quick and helpful from [TPP](https://tpp-uk.com/)
[^3]: As mentioned in the [background](https://github.com/NHS-juju/SFT-Kettering-Work-Log/edit/main/README.md#background), there are potential issues with using `html`, making this an unlikely option.

### Additional Development Work [^4]
[^4]: There was a big delay to beginning this next phase due to other high priority work the Trust was completing.

Following the failure sending `doc`/`docx` to [SystmOne](https://en.wikipedia.org/wiki/SystmOne) additional rollouts were stopped where the document format would be an issue.

The approach at this point is to work on either
1. Engage with suppliers that output `doc`/`docx` and update/enhance their systems to output `pdf`
or
2.Convert `doc`/`docx` to `pdf`.

The decision was made to effectively do both based on the system in play. In the case of the system that produces the vast majority of documents for SFT that outputs `.doc`/`.docx`, they were approached and were able to update their output. For systems where we are only processing a small number of documents per day (between 1 and 50 in a 24 hour period) a local nodejs was developed to convert `doc`/`docx` to `pdf`. rules within the Integration Engine can then be utilised to either do this for all documents, or limit this to those to documents we intend to send to a [SystmOne](https://en.wikipedia.org/wiki/SystmOne) practice.

### Current Position
We have massively improved our position by being able to send documents to GP Practices as a `pdf` which is the original format for the vast majority of the documents being generated, removing any potential issues when converting into `doc`/`docx` and/or into `html`. The work in the integration engine has enabled us to work towards updating all of these routes to `pdf` be it natively or with a one-off conversion using free and open source software. Where the routes pass through old integration engines from one of the old Trusts, this work has also enabled migration works to take place.

### Current Issues
There have been some reports that users in practices are not able to select text from the body of the `pdf` when viewed within [EMISWeb](https://www.emishealth.com/products/emis-web). Tests have confirmed that the text is available within the `pdf` natively (without use of OCR) so this does appear to be related to [EMISWeb](https://www.emishealth.com/products/emis-web) itself. This has been raised with the supplier.
## License
`SFT-Kettering-Work-Log` is licensed under the [Open Government Licence v3.0](./LICENSE) license.
