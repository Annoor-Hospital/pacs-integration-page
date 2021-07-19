This is an article outlining how PACS-integration was implemented on our local OpenMRS + Bahmni based hospital system. It may also help outline how Bahmni pacs-integration works in general. It is also my hope to be able to get feeback about this process, and to hopefully contribute to discussion about how pacs-integration is done in openmrs in general.

## Outline

1. [Hospital Setting](#hospital-setting)
1. [Hospital Systems](#hospital-systems)
1. [PACS Integration Goals](#pacs-integration-goals)
1. [Default Integration](#default-integration)
1. [Customized Integration](#customized-integration)
1. [PACS Processes](#pacs-processes)
1. [Screenshots](#screenshots)

## Hospital Setting

Our local hospital setting is a small hospital for chest diseases / TB sanitorium. The hospital has an outpatient clinic seeing roughly 100 patients per day, as well as approximately 15-20 inpatients on average.

Being a hospital for chest diseases, many patients have a chest X-ray taken. Prior to around 4 years ago, the hospital used a non-digital record keeping system (other than a standalone X-ray storing and viewing station). We have been running a Bahmni-based system since 2017. The hospital has historically had limitted IT staff and support.

## Hospital Systems

When we began EMR integration 4 years ago, we found it helpful to think of at least 3 main digital systems which could be in place.

- The **patient registration** system allows a unique ID to be assigned to patients and stores some basic information about patients. For return patients, it should be possible to search for their previously assigned patient ID.
- The **EMR** system records patient consultation information. This uses the patient's unique ID. Bahmni provides both patient registration and EMR functionality.
- The **PACS/X-ray** system allows X-rays to be taken, stored and retrieved for viewing.

_Other systems include the lab and pharmacy systems, but these aren't included here for berevity._

<img src="/pacs-integration-page/assets/images/pacs-systems.svg" alt="PACS System" style="width:50%;margin:auto;display:block;">

## PACS Integration Goals

Our PACS integration was guided by a number of wishes / goals. Some of the main goals were:

1. If needed, X-rays can be taken without placing an order in the EMR system. This flexability is primarily to have an X-ray procedure even if the EMR system goes offline, or if technical problems arises in the PACS ordering system.
1. The X-ray technician can assign an accession number to X-rays at the modality using a seperate hand-written system. This helps the X-ray system to be able to function without the EMR if needed, and is in keeping with previous procedures.
1. X-ray DICOM files from prior to 2017 should be loaded into the PACS system, and doctors should have the ability to find and view these X-rays.
1. It should be possible for a Radiologist to review X-ray images and provide a radiologist note.

In addition to the goals of the PACS system, integration with the EMR could also provide a number of useful features to streamline hospital workflows. These potential EMR features include:
1. Doctors can order X-rays.
1. Doctors can provide a note for the X-ray technician.
1. X-rays taken by technician can have EMR data pre-filled (for consistent name transliteration, etc).
1. A radiologist can view and attach a radiologist note to the X-ray (within the EMR).
1. Doctors can see a list of X-rays for a given patient.
1. Doctors can view X-rays and see attached radiology notes.

## Default Integration

Some of the above goals are provided by the default Bahmni configuration, but not all. The following outlines the basic structure of the default integration:

<img src="/pacs-integration-page/assets/images/pacs-default.svg" alt="Default PACS Integration" style="width:65%;margin:auto;display:block;">

1. When a doctor orders a PACS image, the order is stored in the OpenMRS database, and an order is sent to the X-ray system.
2. The X-ray order can be fulfilled by the X-ray technician and the X-ray is stored in the PACS system.
3. When the doctor views PACS images for a patient, a list is generated using the orders in the OpenMRS database. Each image has a link generated based on the order number.
4. Once the X-ray order is fulfilled, the link becomes 'live' and will allow the doctor to open and view the X-ray.

Some challenges for us with this method are:
- The order list for a patient is retrieved only from the OpenMRS database. This means that any X-ray images added to the PACS system prior to 2017, or taken without an EMR order, won't be visible.
- The system doesn't know if an order has been fulfilled when the orders are displayed. This is only made known to the doctor when he tries to open an X-Ray order.
- The system requires that the accession number of a new X-Ray match the OpenMRS order number, which deviates from our current accession number system.

For these reasons it seemed necessary to make some customizations to the pacs integration system.

## Customized Integration

A primary modification in our customized integration is to provide an extra functionality in Bahmni to retrieve patient images directly from the PACS system. This is done using a DICOM query, and is used to generate the list of PACS images for a given patient on the patient dashboard.

<img src="/pacs-integration-page/assets/images/pacs-proposal.svg" alt="Proposed PACS Integration" style="width:65%;margin:auto;display:block;">

Rather than only depending on the orders in the OpenMRS database, we query the PACS sytem and match images to their associated OpenMRS order, and combine them into a single list. Then the link to view the X-ray can be generated using the PACS study uuid. This provides a few benefits:
- Bahmni can list all PACS images for a patient, regardless of whether or not they are associated with an order in OpenMRS. This allows old (pre-2017) X-rays to appear in the system, as well as new X-rays taken without an EMR order (in case the EMR system goes offline).
- By querying both PACS and the OpenMRS database, we can see which orders have not been fulfilled and mark them as pending.
- We can avoid using a Bahmni-specific accession number, meaning we can keep our current accession number system.

A second modification of our integration is adding the ability for a radiologist to attach a radiologist note to a PACS image in Bahmni. This is done using a separate Bahmni app (ie link on home screen) where the radiologist can see X-Rays for each day, and enter a note if desires. The note is stored as an observation concept group, containing a reference to the external PACS resoure using the PACS uuid, and the observation note. These notes are then visible to doctors via the Clinical app display control.

The process of retrieving X-Ray orders / images for a patient then involves 3 inputs: OpenMRS pacs orders, external PACS images (queried from the PACS system), and OpenMRS Radiology Notes observations. This is shown in the following diagram:

<img src="/pacs-integration-page/assets/images/combine-order-image-note.svg" alt="Combining Orders and PACS Images" style="width:80%;margin:auto;display:block;">

## PACS Processes

The entire PACS integration system can seem fairly complicated. I will attempt here to outline here the PACS integration system and processes. Note that this system is the same as the default Bahmni integration apart from some customizations and the extra DICOM query in steps 7 and 8.

<img src="/pacs-integration-page/assets/images/pacs-flow.svg" alt="PACS Setup Overview" style="width:100%;margin:auto;display:block;">

1. At the beginning of the process, a doctor places an X-ray order, selecting the appropriate procedure code. The order is then stored with the patient encounter in the OpenMRS database.
2. The OpenMRS atomfeed module publishes the new patient encounter along with the X-ray order. The pacs-integration service continually queries the OpenMRS atomfeed, and when it detects the event it uses the acquired information to construct and send an HL7 order to the PACS system.
4. The PACS system (dcm4chee) converts the HL7 order into a DICOM Modality Worklist item, filling in the DICOM fields using the conversion file `orm2dcm_bahmni.xsl` as configured in DCM4CHEE. The conversion file also assigns a uuid to the DICOM order/study. We construct this uuid as: {organization prefix}.{datetime}.{order number}.
5. The X-ray modality regularly polls the PACS system for MWL items. When the patient arrives at the X-ray station, the X-ray technician selects the MWL order for the patient, which pre-fills a number of fields for the X-ray, such as the Patient ID. In our system, the X-ray technician also enters an X-ray accession number according to a seperate hand-written record.
6. Once the X-ray is taken, it is sent by the Modality via DICOM to the PACS server.
7. The PACS server stores the image, and sends a message to itself using emulated MPPS in order to mark the MWL as completed.
8. The doctor's client Bahmni frontend can be either refreshed or set up to poll for new X-ray images for the patient. Information about patient X-Rays is queried using an OpenMRS module 'pacsquery' which accepts a URL-based request, and performs a corresponding server-side DICOM query. The results are constructed into a JSON response which is sent back to the Bahmni frontend. This response is combined with a query of OpenMRS orders and radiology notes observations. Using these, a list of PACS images and pending orders is created, and for each PACS image a link is created to open the X-ray in a PACS viewer web application. We are using the [OHIF Viewer](https://ohif.org/).
9. Using the Radiology app, a radiologist can view all of the PACS images for a given day. They can open an X-ray for vieweing, and they can also open the patient dashboard to see the patient history and previous PACS images. They can attach a radiologist note to the PACS image to be viewed by the doctors.

## Screenshots

### Link to Radiology App
<img src="/pacs-integration-page/assets/images/screenshot1_home_radiology_link.png" alt="Radiology App Link" style="max-height:400px;margin:auto;display:block;">

### Radiology App: List of daily PACS images
<img src="/pacs-integration-page/assets/images/screenshot5_radiology_app.png" alt="Radiology App List" style="width:100%;margin:auto;display:block;">

### Radiology App: X-Ray Study Notes
<img src="/pacs-integration-page/assets/images/screenshot6_radiology_app2.png" alt="Radiology App Image" style="width:100%;margin:auto;display:block;">

### Patient Dashboard
<img src="/pacs-integration-page/assets/images/screenshot7_pacs_display_control.png" alt="Patient Dashboard" style="width:100%;margin:auto;display:block;">
