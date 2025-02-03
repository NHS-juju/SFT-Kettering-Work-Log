# SFT-Kettering-Work-Log

<a href="https://www.somersetft.nhs.uk/">
	<img alttext="Somerset NHS Foundation Trust Logo" src="https://www.somersetft.nhs.uk/wp-content/uploads/2023/08/Somerset-NHS-logo_EPS-FILE_right-alligned-logo-01-e1690903130748-1024x517.jpg" width="280" />
</a>

# Somerset NHS Foundation Trust - Migrating from Kettering+HTML to Kettering+PDF

> Work logs

## Overview

### Purpose

This repo outlines the steps that have been taken to migrate from Kettering+HTML to Kettering+PDF for sending clinical documents via [MESH](https://digital.nhs.uk/services/message-exchange-for-social-care-and-health-mesh).

### Background

Somerset NHS Foundation Trust has a complex technical estate that generates a vast number of clinial documents for patients under the Trusts care across Somerset. Every encounter with a patient generates a document, usually an inpatient discharge summary, emergency discharge summary, or outpatient clinic letter.

GP Practice Management Systems (PMS) support receiving electronic documents from Secondary Care (and other organisations) using a file type known as `kettering` or `.ket` which is an xml file matching a specific schema. These originally supported a html payload, and specification was later expanded to support a pdf payload. Unfortunately, this change is not well documented, and information on this is difficult to come by. Additionally, at least one PMS supports the payload being `.doc`/`.docx` however this not being the case across all PMS systems authorised for us by the NHS makes this an unfavorable option when attempting to work towards a one-size-fits-all solution.

There are IT systems across the complex technical estate that natively produce PDF, produce Word documents, or even produce html - all of which are received into an Integration Engine. Unfortunately, due to the above mentioned knowledge gap regarding PDFs being supported, the starting point for this work started life with one of the following flows depending on the the system in use and the hospital site it is predominantly used in:
`PDF>HTML>Kettering+HTML`
`Word>HTML>Kettering+HTML`
`PDF>Word>HTML>Kettering+HTML`
`HTML>Kettering+HTML`

This in itself brings a number of issues, mainly
1. Each conversion jump risks the content of the document becoming malformed in some way.
2. There are costs to some software used for the conversions, which add up each year.

Even for instances where the HTML was rendered perfectly or without being converted between different formats, it was not without risks. In years gone by there was a time where there could be major differences in how the HTML would be rendered based on what version of Internet Explorer was being used. There were also cases where receiving systems would [alter the content of the HTML by stripping out various HTML tags (including image tags, removing any images being attached)](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#removed-images-examples) as well as malforming the documents and [removing important content](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#missing-content-examples) or [overlaying paragraphs of text making the content unreadable](https://github.com/TauntonandSomersetNHSTrust/ydh-toc-itk3/tree/main?tab=readme-ov-file#overlapping-paragraph-examples). Due to these issues + others, it was not uncommon for Trusts within a county to avoid sending documents to other counties in case their PMS was from a vendor they had not worked with and these issues exist. However, in the case of Yeovil District Hospital, this was not possible due to being situated so close to the boarder with Dorset, and the examples included in this section highlights the types of issues seen.

### The NHS App, and a renewed interest in improving the status-quo
In October 2023 the Trust was contacted by the [Somerset ICB](https://nhssomerset.nhs.uk/) to see if we could look at what could be done to alter format of documents sent from the Trust to GP Practices within Somerset as sending them in their current format meant that they were present in the PMS EMISWeb as a `.ket` file, which was then not able to be displayed within the NHS App for patients to view.

This led to meetings with product managers from EMISHealth in November 2023 who confirmed that they were able to receive the payloads within the `.ket` files as a BASE64 encoded `.pdf` or `.doc`/`.docx`. Development work was then undertaken by the Trust using the Integration Engine [Intersystems Health Connect](https://www.intersystems.com/uk/products/healthshare/health-connect/).

### Initial Development Work
The basis of the technical work actually came from example code that was shared with the Trust by contacts at Intersystems UK. Part of this code has a message class that, once populated and saved, produces the `XML` portion of the `.ket` as well as adding in the payload. What was interesting is that some commwents within the code indicates that the ability to have a `PDF` payload was added in sometime in 2019. As this work was being done rather myopically, the revelation that not all PMS's support anything more than `HTML` or `PDF` payloads, work was done to extend this codebase to also support `.doc`/`.docx`.

Once an example output could be produced, this was shared with contacts at EMISHealth to test the processing of the generated files from the Integration Engine. This was found to have been successful, so the next steps were to begin transitioning to the new setup. This consisted of updating the existing routes within the Integration Engine [Intersystems Health Connect](https://www.intersystems.com/uk/products/healthshare/health-connect/) to build the messages into a common message with either a `.pdf` or `.doc`/`.docx` payload and then use this to then output the files required for outputting to the GP systems [via MESH](https://digital.nhs.uk/services/message-exchange-for-social-care-and-health-mesh)

### Path-To-Live (Attempt 1)
The plan for moving forward with the works to move from the existing setup was simple:
1. Deploy the change for `system x` output into the Live environment
2. Contact a number of GP practices and confirm that they are able to view the documents we have sent
3. Rinse and repeat.

The first few systems attempted with this were outputting `.pdf` files, which meant that the tests were successful. However, the first tests with a system outputting `.doc`/`.docx` failed when checking with a practice using [TPP](https://en.wikipedia.org/wiki/SystmOne) and a quick confirmation from their support team confirmed [^1] that they only support Kettering+HTML[^2] or Kettering+PDF . Therefore the change for this system was rolled back.
[^1]: It's sometimes dofficutl engaging with suppliers in this context as we're not their customer, so this was achieved by asking the affected practice to contact the supplier on my behalf. Thankfully, the response was very quick and helpful.
[^2]: As mentioned in the [background](https://github.com/NHS-juju/SFT-Kettering-Work-Log/edit/main/README.md#background), there are potential issues with using HTML, making this an unlikely option.

### Additional Development Work [^3]
[^3]: there was a big delay to beginning this next phase due to other high priority work the Trust was completing.
Following the failure sending `.doc`/`.docx` to [TPP](https://en.wikipedia.org/wiki/SystmOne) additional rollouts were stopped where the document format would be an issue.

The approach at this point is to work on either
1. Engage with suppliers that output `.doc`/`.docx` and update/enhance their systems to output `.pdf`
or
2.Convert `.doc`/`.docx` to `.pdf`.

The decision was made to effectively do both based on the system in play. In the case of the system that produces the vast majority of documents for SFT that outputs `.doc`/`.docx`, they were approached and were able to update their output. For systems where we are only processing a small number of documents per day (between 1 and 50 in a 24 hour period) a local nodejs was developed to convert `.doc`/`.docx` to `.pdf`. rules within the Integration Engine can then be utilised to either do this for all documents, or limit this to those to documents we intend to send to a [TPP](https://en.wikipedia.org/wiki/SystmOne) practice.

### Current Position
todo

### Current Issues
todo
## License
`SFT-Kettering-Work-Log` is licensed under the [Open Government Licence v3.0](./LICENSE) license.
