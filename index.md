Text goes here about what the purpose of this article is.

## Outline

1. [Hospital Setting](#hospital-setting)
2. [Integration Goals](#integration-goals)
3. [Integration Proposal](#integration-proposal)
4. [Visual Demonstration](#visual-demonstration)

## Hospital Setting

- small hospital / TB sanitorium
  - 100 outpatients per day
  - 20 inpatients
  - Being a hospital for chest diseases, many of the outpatients have a chest x-ray taken
- recently migrated from a non-digital system (apart from a standalone X-Ray store / viewing system) to an EMR system
- have been running a Bahmni-based system since 2017
- limitted IT staff and support.

## Integration Goals

- three primary systems: patient registration, EMR, pacs.
  - also lab orders / tests, pharmacy orders and stock management (which includes billing for things like X-Ray)
  - focus of this article is not lab / pharmacy system

<img src="/pacs-integration-page/assets/images/pacs-systems.svg" alt="PACS System" style="width:50%;margin:auto;display:block;">

1. patient registration: assigning a reasonable ID to a patient. When a patient is a return patient, being able to find him in the system.
2. EMR: vital signs, observations and diagnosis. Also includes lab orders, pharmacy orders, etc.
3. X-Ray / PACS: taking X-Rays and searching for & viewing old X-Rays

### Goals for X-Ray / PACS Integration:
- X-Rays can be taken if needed without placing an order in the EMR system. This flexability is primarily to have an X-Ray procedure that will work even if the EMR system goes offline, or if technical problems arises in the PACS ordering system.
- The X-Ray technician can assign an accession number to X-Rays at the modality (these are recorded by hand). This is related to the previous goal, and is in keeping with practice prior to installing the EMR system. This accession number can at times aid doctors in searching for past X-Rays.
- X-Ray DICOM files from prior to 2017 should be loaded into the PACS system, and doctors should have the ability to find and view these X-Rays.
- It must be possible for a Radiologist to review X-Ray images and provide a radiologist note.
- EMR (Bahmni) Integration. The following are helpful integration features which can streamline doctor's work process:
  - Doctors can order X-Rays
  - Doctors can provide a note for the X-Ray technician.
  - Radiologist gets notified of new X-Rays
  - Radiologist can digitally attach a note to the X-Ray
  - Doctors can see a list of X-Rays for a given patient
  - Doctors can view the x-ray and the attached radiology notes.

## Integration Proposal

### Default Bahmni integration

A number of the above goals are provided by the default Bahmni pacs-integration system.

A simple outline of my understanding of the default Bahmni pacs-integration logic:

<img src="/pacs-integration-page/assets/images/pacs-default.svg" alt="Default PACS Integration" style="width:65%;margin:auto;display:block;">

Steps:
- When a doctor orders a PACS image, the order is stored in the OpenMRS database, and an order is sent to the X-Ray system (dcm4chee, then modality).
- When the doctor lists PACS images for a patient, this list is generated using the orders in the OpenMRS database. Each image has an attached link based on the order number.
- The X-Ray order is fulfilled by the X-Ray technician and stored in the PACS system.
- Once the X-Ray is fulfilled, the link becomes 'live' and will allow the doctor to open and view the X-Ray.

Primary issues:
- Bahmni uses the DICOM accession number field to store the PACS order number.
- The only X-Rays visible in the EMR system are those which have a corresponding EMR order. X-Rays taken prior to EMR deployment, and those taken without using the EMR system will not be visible on the patient dashboard.

### Customized Integration

One modification of this system is to provide a way for Bahmni to directly query the PACS system for X-Rays of a given patient.

<img src="/pacs-integration-page/assets/images/pacs-proposal.svg" alt="Proposed PACS Integration" style="width:65%;margin:auto;display:block;">

Replace the pacs display control. The PACS list is built by combining a query for OpenMRS orders and a DICOM query of pacs images.
- Orders without a corresponding image are considered unfulfilled orders. No view link is provided for these orders, and the doctor has an option to cancel the order.
- PACS images are displayed whether or not a corresponding OpenMRS order exists. The only thing necessary is that the patient ID matches. For each PACS image, a viewing link is provided based on the DICOM uuid.

In addition, add functionality for a radiologist to attach a note to a PACS image.
- concept group corresponding to the external resource (PACS image) containing note, as well as uuid of PACS image
- pacs display control queries for notes corresponding to the pacs images, and makes the note visible to doctors from the patient dashboard.

This process involves 3 inputs: OpenMRS pacs orders, PACS images, and OpenMRS Radiology Notes observations.

<img src="/pacs-integration-page/assets/images/combine-order-image-note.svg" alt="Combining Orders and PACS Images" style="width:80%;margin:auto;display:block;">


### PACS order and viewing process
The entire process of ordering & viewing X-Rays can seem quite complicated. I will here outline our integration system as I understand it. Note that this is the same as the default Bahmni integration, apart from some customizations and the extra DICOM query in steps 7 and 8.

<img src="/pacs-integration-page/assets/images/pacs-flow.svg" alt="PACS Setup Overview" style="width:100%;margin:auto;display:block;">

1. Doctor selects a procedure code and places an X-Ray order. A note can be added for the X-Ray technician. The order is stored along with a patient encounter in the OpenMRS database.
2. The pacs-integration service queries the OpenMRS atomfeed, detecting the encounter event and the X-Ray order. It uses the acquired information to construct and send an HL7 order to the PACS system.
3. The PACS system (dcm4chee) converts the HL7 order into a Modality Worklist item, filling in the DICOM fields using the conversion file `orm2dcm_bahmni.xsl` as configured in DCM4CHEE. The conversion file also assigns a uuid to the DICOM order/study. We construct this as: {organization prefix}.{datetime}.{order number}.
4. The X-Ray modality is configured to regularly poll the PACS system for MWL items. When the patient arrives at the X-Ray station, the X-Ray technician selects the MWL order for the patient. This pre-fills a number of fields for the X-Ray, such as the Patient ID. The X-Ray technician also fills out the X-Ray accession number using a seperate hand-written record.
5. Once the X-Ray is taken, it is send via DICOM to the PACS server.
6. The PACS server stores the image, and sends a message to itself using emulated MPPS in order to mark the MWL as completed.
7. The doctor's client Bahmni frontend can be either refreshed or set up to poll for new X-Ray images for the patient. This is done using a small OpenMRS module which gets a URL-based PACS query, then performs a server-side DICOM query, and uses the results to construct a JSON response to be sent back to Bahmni. The PACS iamges are listed along with a link to view them in a web-based PACS viewer. Radiology Notes observations are also queried from the OpenMRS DB to be shown alongside X-Rays.
8. A radiologist makes use of a Bahmni app to query and view all of the X-Rays for a given day (today). He/she can  view the X-Ray and has the option to view the patient dashboard in Bahmni, and can attach a Radiology Notes observation to the X-Ray to be viewed by the doctors.

## Visual Demonstration

etc