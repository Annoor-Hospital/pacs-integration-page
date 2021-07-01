This is an article outlining how PACS-integration was implemented on our local OpenMRS + Bahmni based hospital system. It may also help outline how Bahmni pacs-integration works in general. It is also my hope to be able to get feeback about this process, and to hopefully foster discussion about how pacs-integration can be better supported in openmrs in general.

## Outline

1. [Hospital Setting](#hospital-setting)
2. [Integration Goals](#integration-goals)
3. [Integration](#integration)
4. [Visual Demonstration](#visual-demonstration)

## Hospital Setting

Our local hospital setting is a small hospital for chest diseases / TB sanitorium. The hospital has an outpatient clinic seeing roughly 100 patients per day, as well as approximately 15-20 inpatients on average.

Being a hospital for chest diseases, many patients have a chest X-ray taken. Prior to around 4 years ago, the hospital used a non-digital record keeping system apart from a standalone X-ray storing and viewing station. We have been migrating to using electronic records, and been running a Bahmni-based system since 2017. The hospital has historically had limitted IT staff and support.

## Hospital Systems

When we began EMR integration 4 years ago, we found it helpful to think of at least 3 main digital systems which could be in place.

One system is *patient registration*, allowing a unique ID to be assigned to patients. When a patient is a return patient, it should be possible to search for their previously assigned patient ID.

Another system is the *EMR* system, for recording patient consultation information. These should be attached to the given patient ID. Bahmni provides both patient registration and EMR functionality.

A third system is the *PACS/X-ray* system, allowing X-rays to be taken, stored and later retrieved.

_Other systems include the lab and pharmacy systems, but these aren't included here for berevity._

<img src="/pacs-integration-page/assets/images/pacs-systems.svg" alt="PACS System" style="width:50%;margin:auto;display:block;">

### PACS Integration Goals

Our PACS integration was guided by a number of wishes / goals. Some of the main goals were:

1. X-rays can be taken if needed without placing an order in the EMR system. This flexability is primarily to have an X-ray procedure that will work even if the EMR system goes offline, or if technical problems arises in the PACS ordering system.
1. The X-ray technician can assign an accession number to X-rays at the modality according to a seperate hand-written record. This allows the X-ray system to be able to function if needed without the EMR, and is in keeping with previous X-ray procedures.
1. X-ray DICOM files from prior to 2017 should be loaded into the PACS system, and doctors should have the ability to find and view these X-rays.
1. It must be possible for a Radiologist to review X-ray images and provide a radiologist note.

In addition to the goals of the PACS system, integration with the EMR could provide a number of useful features to streamline hospital workflows. These potential EMR features include:
1. Doctors can order X-rays
1. Doctors can provide a note for the X-ray technician.
1. X-rays taken by technician can have EMR data pre-filled (ie for consistent name transliteration).
1. Radiologist can view and digitally attach a radiologist note to the X-ray within the EMR
1. Doctors can see a list of X-rays for a given patient
1. Doctors can view the X-ray and the attached radiology notes

## Default Bahmni pacs-integration

Some of the above goals are provided by the default Bahmni configuration, but not all. The following outlines the basic structure of the default integration:

<img src="/pacs-integration-page/assets/images/pacs-default.svg" alt="Default PACS Integration" style="width:65%;margin:auto;display:block;">

1. When a doctor orders a PACS image, the order is stored in the OpenMRS database, and an order is sent to the X-ray system.
2. The X-ray order can be fulfilled by the X-ray technician and the X-ray is stored in the PACS system.
3. When the doctor views PACS images for a patient, a list is generated using the orders in the OpenMRS database. Each image has a link generated based on the order number.
4. Once the X-ray order is fulfilled, the link becomes 'live' and will allow the doctor to open and view the X-ray.

Some challenges for us with this method are:
- The order list for a patient is retrieved only from the OpenMRS database. This means that any X-ray images added to the PACS system but not associated with a Bahmni order will not be visible.
- The system does not know if an order has been fulfilled or not when the orders are displayed. This is only made known to the doctor when he tries to open an X-Ray order.
- The setup requires that the accession number of a new X-Ray match the OpenMRS order number, which deviates from our current system.

## Customized Integration

One modification of this system is to provide a way for Bahmni to directly query the PACS system for X-rays of a given patient.

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
The entire process of ordering & viewing X-rays can seem quite complicated. I will here outline our integration system as I understand it. Note that this is the same as the default Bahmni integration, apart from some customizations and the extra DICOM query in steps 7 and 8.

<img src="/pacs-integration-page/assets/images/pacs-flow.svg" alt="PACS Setup Overview" style="width:100%;margin:auto;display:block;">

1. Doctor selects a procedure code and places an X-ray order. The order is stored along with a patient encounter in the OpenMRS database.
2. The pacs-integration service queries the OpenMRS atomfeed, detecting the encounter event and the X-ray order. It uses the acquired information to construct and send an HL7 order to the PACS system.
3. The PACS system (dcm4chee) converts the HL7 order into a Modality Worklist item, filling in the DICOM fields using the conversion file `orm2dcm_bahmni.xsl` as configured in DCM4CHEE. The conversion file also assigns a uuid to the DICOM order/study. We construct this as: {organization prefix}.{datetime}.{order number}.
4. The X-ray modality is configured to regularly poll the PACS system for MWL items. When the patient arrives at the X-ray station, the X-ray technician selects the MWL order for the patient. This pre-fills a number of fields for the X-ray, such as the Patient ID. The X-ray technician also enters an X-ray accession number according to a seperate hand-written record.
5. Once the X-ray is taken, it is sent via DICOM to the PACS server.
6. The PACS server stores the image, and sends a message to itself using emulated MPPS in order to mark the MWL as completed.
7. The doctor's client Bahmni frontend can be either refreshed or set up to poll for new X-ray images for the patient. This is done using a small OpenMRS module which accepts a URL-based PACS query, then performs a coresponding server-side DICOM query. It uses the results to construct a JSON response to be sent back to the Bahmni frontend. A list of PACS iamges is constructed along with links to view images in a web-based PACS viewer. Radiology Notes observations are also queried from the OpenMRS DB to be shown alongside X-rays.
8. A radiologist makes use of a bahmni Radiology app to query and view all of the X-rays for the given day. He/she can open the X-ray for viewing and has the option to view the patient dashboard in Bahmni, making it easy to see the patient history and previous X-rays. He/she can attach a radiologist note to the X-ray to be viewed by the doctors.

## Visual Demonstration

etc