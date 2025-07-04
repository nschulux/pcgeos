# 17 The Spool Library
Any geode can work with printers, fax machines, and other print devices by 
including the spool library and instantiating a PrintControl object to handle 
system interactions. The **PrintControlClass** provides the printing 
interface. It includes both user interface and an interface between the geode 
and the printing thread. There is also a **PageSizeControlClass** which 
provides UI concerning page layout.

Before reading this chapter, you should be familiar with the graphics system.

## 17.1 Introduction to Printing
By including a PrintControl object, an application gains access to the powers 
of the GEOS printing system. Below are several of the features built into 
GEOS:

+ Single Imaging Model  
You'll use the same commands to describe print jobs as you did for screen 
drawings. These commands work for all supported printers. Thus, 
anything you can draw can be printed, and the application doesn't have 
to worry about what model of printer is being used.

+ Background Printing  
Background printing allows users to continue working while their 
documents print. This is an example of multitasking at work. A process 
known as the Spooler spawns a separate thread for each active printer. 
Since each printer's thread priority is low, printing will take place in the 
background, interfering little with the user's interactions. If you want 
printing to take place in the foreground, your application can boost the 
priority of a printer thread.

+ Standard UI Components  
The PrintControl is a full-featured control object and it provides a UI 
mechanism for finding out what a user wants to print. For user choices 
which depend on the printer (for example, some printers have no 
low-quality mode), the PrintControl will take printer abilities into 
consideration. (See Figure 17-1.)  
![image info](Figures/Fig17-1.png)  
**Figure 17-1** *Print Dialog Box*  
*This Print dialog box shows a Print Control plus some 
specialized gadgets supplied by an application.* 

+ Control Over Scheduling  
The Spooler normally handles jobs in FIFO (first in, first out) order. 
Geodes can exercise a great deal of control over the Spooler's scheduling 
of jobs. Any scheduling options the user might request using the Printer 
Control Panel (see Figure 17-2), the application can request using spool 
routines.  
![image info](Figures/Fig17-2.png)  
**Figure 17-2** *Printer Control Panel*

+ Managing the Printer Yourself  
If you can't or won't use standard GEOS commands to describe a print job, 
you can use Raw Mode to send a packet of commands in the printer's own 
language. For example, you could send escape codes or commands from a 
page description language. Since this is a standard printing mode, you 
still benefit from all the usual spool scheduling features.

+ Management of Multiple Printers  
The user may have multiple printing devices hooked up and installed at 
one time. The system handles this situation intelligently: the Spooler 
maintains a separate queue for each serial or parallel port. Thus, if two 
printers are installed on two different ports, both may print at once. If 
two or more printers are installed on the same port then they share a 
queue (see Figure 17-3). Thus, jobs for one printer won't try to print at 
the same time as those for another on the same port. To make sure the 
job prints to the correct printer, the Spooler will put up a dialog box 
advising the user to put the correct device on line.  
![image info](Figures/Fig17-3.png)  
**Figure 17-3** *Multiple Printer Queues*  
*The Spooler maintains separate queues for separate ports. If all 
printers shared the same queue, the incoming job C wouldn't be 
able to print until after jobs A and B had finished. Instead, job 
C will be the next thing printed on its device.*

## 17.2 Simple Printing Example
Adding printing capability to an application is fairly simple, especially if it 
will be printing WYSIWYG. Somewhere in the application there is probably a 
message handler which is in charge of drawing the document. By including a 
Print Control object and writing handlers to some standard messages, it is a 
simple manner to redirect the document drawing commands to a printer.

----------
**Code Display 17-1 Printing Example**

    @object GenApplicationClass MyApp = {
        /* -Much other GenApplication instance data omitted- */
        gcnList(MANUFACTURER_ID_GEOWORKS, GAGCNLT_SELF_LOAD_OPTIONS) = 
            @MyPrintControl;
        ATTR_GEN_APPLICATION_PRINT_CONTROL = @MyPrintControl; 
    }

    @object PrintControlClass MyPrintControl = {
    /* The PrintControlClass is a subclass of GenControlClass. Its "features"
     * correspond to the "Print" and "Fax" triggers in the File menu. Its instance
     * data contains information about the document to be printed, and links to
     * objects which will provide information. */

    /* Single page documents would leave out page range UI from the Print dialog: 
        PCI_attrs =         (@default & ~(PCA_PAGE_CONTROLS| PCA_VERIFY_PRINT)); 
        PCI_startUserPage = 1;
        PCI_endUserPage = 1 */

    /* The PCI_output, or Print Output object, is in charge of supplying the graphics
     * commands describing the print job whenever the user wants to print. This object
     * is expected to handle a MSG_PRINT_CONTROL_START_PRINTING. */ 

     * The print output object is normally either the model, target or process. */
        PCI_output =        TO_APP_TARGET;

    /* The PCI_docNameOutput object is supposed to provide the name of the document
     * on demand. This name is used to identify the document to the user in the Print
     * Control Panel. If this object is a GenDocumentGroup, it automatically does this.
     * If the object in the following field is not a GenDocumentGroup object, then
     * it must have a handler for MSG_PRINT_GET_DOC_NAME. */
        PCI_docNameOutput = MyGenDocumentGroup;

    /* Many simple applications only support one document size,
     * and may specify it in the PrintControl and never bother with it again: */
        PCI_docSizeInfo = {         (15/2*72), 
                        (19/2*72), 
                        0, 
                        {0, 0, 0, 0}}; */

    /* Many simple applications do not support multiple documents. These applications 
     * should set the GS_ENABLED flag of the PrintControl's GI_states field. */
    }

    @method MVTStartPrinting, MyVisTargetedClass, MSG_PRINT_START_PRINTING
    /* We've set up our Print Control to send this message to the Target, so whatever
     * object will have the target when the user wants to print needs a handler for 
     * this message. We could have easily have put the Model or process in charge
     * of printing, changing PCI_output accordingly. It is often convenient to use
     * the same object to handle drawing to screen and to the printer. */

    /* Arguments:       optr        printControlOD,
                        GStateHandle        gstate */

    {
        PCMarginParams  margins;

    /*
     *  Applications which allow the user to change the document size may handle
     *  the situation in more than one way. If the user has changed the page setup
     *  by working with a PageSizeControl, then the application has probably 
     *  already been alerted to the page size. If the application has its own
     *  way of computing document size, it should use it. Such applications should
     *  send MSG_PRINT_CONTROL_SET_DOC_SIZE and MSG_PRINT_CONTROL_SET_DOC_MARGINS 
     *  to the PrintControl, either in this handler or else whenever the page size
     *  changes. Either time is fine, just so long as the document size is set
     *  correctly by the time this MSG_PRINT_START_PRINTING handler is finished.

     *  Applications supporting only a single document size should probably set
     *  the size (and margins) in the PrintControl's instance data, as shown
     *  above. It is also possible to send a MSG_PRINT_CONTROL_SET_DOC_SIZE on
     *  every print, but if the size never changes, this is a bit wasteful.

     *  This application supports only one size of document. If the document
     *  doesn't fit on the page, the spooler will tile it onto multiple pages
     *  as necessary. It uses the printer's margins as the document margins: */

        @call   MyPrintControl::MSG_PRINT_CONTROL_GET_PRINTER_MARGINS(
        &margins,   /* Fill in structure with printer's margins */
        TRUE);      /* ...and automatically use printer margins
                     * as document margins */

        @call   self::MSG_VIS_DRAW(DF_PRINT, gstate);

        /* If the MSG_VIS_DRAW handler didn't end with a form feed, put one in: 
        GrNewPage(gstate, PEC_FORM_FEED); */

        @send   MyPrintControl::MSG_PRINT_CONTROL_PRINTING_COMPLETED();
    }

    @method MVTDraw, MyVisTargetClass, MSG_VIS_DRAW
    /* Chances are, whatever object this is has some sort of draw handler already.
     * However, if you're building up this class from scratch, you'll be glad to
     * know that commands of the following form work just as well drawing in
     * response to a print request as they do drawing anything else: */

    {   GrSetLineColor(gstate, CF_INDEX, C_RED, 0, 0);
        GrDrawLine(gstate, 144, 144, 288, 288);
        GrNewPage(gstate, PEC_FORM_FEED);
    }

----------
## 17.3 How Jobs Get Printed
You may wonder how and when these various features get implemented. 
Simply stated, the application describes the print job, which gets placed onto 
a queue where it remains until it is fed to a printer. For many programmers, 
that's enough to know. If you want a more complete explanation, read on.

### 17.3.1 Printing System Components
Printing involves the concerted effort of several objects running in different 
threads. The most important pieces of the printing system are

**PrintControl**  
The printing system is big and would be difficult to interact 
with if not for the Print Control, which acts as a sort of 
intermediary. This object is a member of the 
**PrintControlClass**, a GenControl subclass provided by the 
spool library. It handles the UI, communication with the spool 
thread, and most other printing functions. Every geode which 
is going to print includes its own Print Control, placing the 
control object in its generic tree.

**Print Output**  
The Print Output is an object (often the process object) chosen 
by the geode when creating the Print Control. This object is in 
charge of building print jobs and must be prepared to provide 
page descriptions whenever it receives a 
MSG_PRINT_START_PRINTING from the Print Control.

**Spooler**  
The Spooler handles most of the important "behind the scenes" 
work of printing. As mentioned, it handles the scheduling of 
jobs and manages the operations by which generic print jobs 
are translated for specific printers.

**Printer Driver**  
GEOS printer drivers handle the back end of printing. GEOS 
has many printer drivers, each of which serves one or more 
models of printer. The driver supplies the Spooler with 
information about the printer and makes some final 
adjustments to the translated print job to get it ready for the 
specific printer.

### 17.3.2 Chronology
Now that you're somewhat familiar with these components, you're ready for 
the detailed account of how they work together to print a job. Note that the 
application is only involved at one point. The printing system takes care of 
the rest automatically.

1. **User Activates Trigger**  
Printing normally begins when the user activates the User Print Trigger 
provided by the Print Control (see Figure 17-4). This trigger sends a 
message to the Print Control telling it to put up the Print Dialog box.  
![image info](Figures/Fig17-4.png)  
**Figure 17-4** *User Print Trigger*  
*In some specific user interfaces, the standard User Print Trigger is an 
entry in the File Menu.* 

2. **User Interacts with the Print Dialog Box**  
The Print Control responds to the Print Trigger's message by presenting 
a dialog box in which the user selects such things as page range and print 
quality (see Figure 17-1 above). The user then either initiates 
printing or cancels. If the user wants to print, he'll confirm this by 
clicking on the appropriate trigger. The application will then get a chance 
to veto the user's choices. The Print Control asks the dialog box to remove 
itself, and it then gets ready for the upcoming print job.

3. **Print Control Prepares for New Job**  
Soon the Print Control is going to ask for the page descriptions, but first 
it needs a place to store the data. It creates a spool file in which to store 
the upcoming graphics commands. It also allocates a GString handle by 
which the Print Output can transmit the commands. It then sends a 
message to the application's Print Output, signalling that the Print 
Output should supply the job. (See Figure 17-5.)  
![image info](Figures/Fig17-5.png)  
**Figure 17-5** *Preparing for New Print Job*  
*The Print Control readies a file to hold data and alerts the application*

4. **Application Supplies Job**  
Having received the Print Control's message, the Print Output supplies 
graphics routines to the provided GString handle, using the same 
commands as when printing to any other GState. This string of graphics 
commands ends with a message saying the job is completed. The Print 
Control supplies some extra information to the spooler, and the print job 
is ready to enter the queue. (See Figure 17-6.)  
![image info](Figures/Fig17-6.png)  
**Figure 17-6** *Application Supplies Job*  
*The application supplies a page description, which goes through the 
GString handle supplied by the Print Control into the Spool File.*

5. **Job Traverses Queue**  
At this point, the spool library places the print job on a printer queue 
where the job waits to be printed. (Actually, the Spooler uses a trick to 
save on overhead. If there are no jobs for a printer, the Spooler doesn't 
maintain a queue for that printer. When a new job comes in for a printer 
that has no queue so far, a new queue is prepared for that printer and the 
job placed on the new queue. When the last job of a queue is printed and 
there are no more jobs forthcoming, the queue is removed.) The Spooler's 
various scheduling powers, if exercised, affect jobs in the queue. (See 
Figure 17-7.)  
![image info](Figures/Fig17-7.png)  
**Figure 17-7** *Job Traverses Queue*

6. **Printing**  
Eventually, the job reaches the head of the queue. It is now ready to print. 
The Spooler works with the Printer Driver to transform the print job into 
a form that the printer can work with. This may involve building a 
bitmap depicting the job, or it might involve translating the GString's 
commands into the commands of some other page description language, 
such as PostScript. (See Figure 17-8.)  
![image info](Figures/Fig17-8.png)  
**Figure 17-8** *Printing*  
*The Spooler converts the print job to a format the printer can understand.*

7. **Hard Copy**  
The printer driver sends the appropriate printer commands to the 
printer. If the printer doesn't have sufficient memory to hold the whole 
job, the driver sends only part of the job at a time.

## 17.4 Print Control Instance Data
The PrintControl is a powerful and adaptable subclass of GenControl and 
contains many instance fields to fit the needs of different types of 
applications. Each field is described in greater detail later in this section; all 
are shown in Code Display 17-2.

----------
**Code Display 17-2 Print Control Instance Data and Features**

        /* The following bitfield contains the PrintControl's attributes */
    @instance PrintControlAttrs         PCI_attrs = 
                (PCA_COPY_CONTROLS | PCA_PAGE_CONTROLS |
                 PCA_QUALITY_CONTROLS | PCA_USES_DIALOG_BOX |
                 PCA_GRAPHICS_MODE | PCA_TEXT_MODE );
        /* Possible PCI_attrs flags (may be combined using | and &):
         * PCA_MARK_APP_BUSY,               PCA_VERIFY_PRINT,
         * PCA_SHOW_PROGRESS,               PCA_PROGRESS_PERCENT, 
         * PCA_PROGRESS_PAGE,               PCA_FORCE_ROTATION 
         * PCA_COPY_CONTROLS,               PCA_PAGE_CONTROLS,
         * PCA_QUALITY_CONTROLS,                PCA_USES_DIALOG_BOX,
         * PCA_GRAPHICS_MODE,               PCA_TEXT_MODE 
         * PCA_DEFAULT_QUALITY                      */

        /* The fields below are the complete and requested page ranges */
    @instance word      PCI_startPage = 1;
    @instance word      PCI_endPage = 1;
    @instance word      PCI_startUserPage = 0;
    @instance word      PCI_endUserPage = 0x7fff";

        /* The following field contains the default printer number */
    @instance word      PCI_defPrinter = -1;

        /* The following fields deal with document dimensions */
    @instance PageSizeReport        PCI_docSizeInfo = 0;

        /* Pointers to objects receiving vital messages */
    @instance optr      PCI_output;
    @instance optr      PCI_docNameOutput;

        /* The PrintControl's features determine whether to use standard
         * print and fax triggers to bring up the Print Dialog box. */
    typedef ByteFlags PrintControlFeatures;
    /* The following flags may be combined with | and &:
        PRINTCF_PRINT_TRIGGER,
        PRINTCF_FAX_TRIGGER */
    /* To provide a non-standard print trigger, use the GenControl vardata
     * field ATTR_GEN_CONTROL_APP_UI. */
    typedef ByteFlags PrintControlToolboxFeatures;
    /* The following flags may be combined with | and &:
        PRINTCTF_PRINT_TRIGGER 
        PRINTCTF_FAX_TRIGGER */

    /* To include application-specific UI in the print dialog box. */
    @vardata optr ATTR_PRINT_CONTROL_APP_UI;

    /* This piece of temporary vardata is internal: */
    @vardata TempPrintCtrlInstance TEMP_PRINT_CONTROL_INSTANCE;

    /* Its structures are defined as follows:
    typedef struct {
         optr           TPCI_currentSummons; ( currently active summons )
         optr           TPCI_progressBox; ( OD of progress dialog box */
         ChunkHandle    TPCI_jobParHandle; ( handle to JobParamters )
         word           TPCI_fileHandle; ( file handle (if printing) )
         word           TPCI_gstringHandle; ( gstring handle if printing )
         word           TPCI_printBlockHan; ( the printer block handle )
         PrintControlAttrs          TPCI_attrs;
         PrintStatusFlags           TPCI_status;
         byte           TPCI_holdUpCompletionCount;
    } TempPrintCtrlInstance;
    typedef ByteFlags PrintStatusFlags;
    #define PSF_FAX_AVAILABLE 0x80 ( set if a fax driver is available )
    #define PSF_ABORT 0x08 ( user wants to abort printing )
    #define PSF_RECEIVED_COMPLETED 0x04 ( MSG_-_PRINTING_COMPLETED received )
    #define PSF_RECEIVED_NAME 0x02 ( MSG_PC_SET_DOC_NAME received )
    #define PSF_VERIFIED 0x01 ( PSG_PC_VERIFY_? received ) */

    @vardata TempPrintCompletionEventData TEMP_PRINT_COMPLETION_EVENT;

    /* The TempPrintCompletionEventData structure is defined:
    typedef struct {
         MemHandle      TPCED_event;
         MessageFlags       TPCED_messageFlags;
    } TempPrintCompletionEventData; 

----------
### 17.4.1 Alerting the GenApplication
PrintControl objects should be placed on the GenApplication's 
MANUFACTURER_ID_GEOWORKS/GAGCNLT_SELF_LOAD_OPTIONS general 
change notification list. Also, if the application is to allow printing of 
documents from the file from the file manager, then the PrintControl's optr 
should be specified in the GenApplication's 
ATTR_GEN_APPLICATION_PRINT_CONTROL field.

### 17.4.2 Attributes
    PCI_attrs, MSG_PRINT_CONTROL_SET_ATTRS, 
    MSG_PRINT_CONTROL_GET_ATTRS

The Print Control includes several attributes which are grouped together 
into a record of type **PrintControlAttrs**. These represent some choices 
made when instantiating the Print Control.

PCA_MARK_APP_BUSY  
Describing large print jobs can take a fair amount of time. If 
this bit is set, the application will reassure the user that it is 
busy (by showing the busy cursor) while spooling long print 
jobs.

PCA_VERIFY_PRINT  
If this bit is set, the Print Control will ask the application for 
confirmation before printing anything. This request will come 
in the form of a MSG_PRINT_VERIFY_PRINT_REQUEST sent to 
the PCI_output object. The Print Control sends this message 
after the user has finished interacting with the Print dialog box 
so that the application can make sure that the user has made 
valid choices.

PCA_SHOW_PROGRESS, PCA_PROGRESS_PERCENT, PCA_PROGRESS_PAGE  
When describing a large print job, it is considerate to let the 
user know when the application is making some kind of 
progress. If the first of these bits is set, the Print Control will 
display a progress dialog box when spooling a print job. The 
next two bits determine whether progress will be reported as a 
percentage, number of pages completed, both, or neither. 
Regardless of whether percents or page number progress is 
reported, the progress box can display an application-supplied 
string. 

PCA_FORCE_ROTATION  
If this bit is set, the output will automatically be printed 
rotated on the page. Normally, this bit is not set so the Spooler 
will make a choice to make the job fit on the minimum number 
of pages. The Banner program sets this bit to make sure that 
fanfold paper printers will create continuous banners; if it 
didn't print sideways, then users would have to use a lot of tape 
to put their banners together (they have to anyhow, if using non 
continuous-feed paper).

PCA_COPY_CONTROLS  
This flag determines whether the Print Control will allow the 
user to request that multiple copies be printed. If your 
application uses this option, then it is vital that the print 
control be updated whenever the total page count of the 
document changes.

PCA_PAGE_CONTROLS  
If this bit is set, the user will be allowed to select specific page 
ranges to print. Even if it seems like users would have little 
reason to print short ranges, it's still nice to include this option: 
if the printer messes up one page, it's usually irritating to have 
to reprint the whole document.

PCA_QUALITY_CONTROLS  
If this flag is on, the user will be allowed to select which quality 
to print with. Note that not all printers can print at all 
qualities, but most support more than one mode.

PCA_USES_DIALOG_BOX  
This bit determines whether the Print Control will display a 
dialog box containing printing choices for the user. Specialty 
geodes and GCM applications might want to turn this flag off; 
it is set by default.

PCA_GRAPHICS_MODE, PCA_TEXT_MODE  
These bits determine whether the Control should support 
graphic mode and/or text mode output.

PCA_DEFAULT_QUALITY  
These two bits contain the default print quality to use. The 
enumerated type **PrintQualityEnum** defines the values 
available: PQT_HIGH, PQT_MEDIUM, and PQT_LOW.

Those attributes which are on by default are PCA_COPY_CONTROLS, 
PCA_PAGE_CONTROLS, PCA_QUALITY_CONTROLS, 
PCA_USES_DIALOG_BOX, PCA_GRAPHICS_OUTPUT, and 
PCA_TEXT_OUTPUT.

There are messages to get and set these attributes.

----------
#### MSG_PRINT_CONTROL_SET_ATTRS
    void    MSG_PRINT_CONTROL_SET_ATTRS(
            PrintControlAttrs   attributes);

Use this message to change the values stored in the *PCI_attrs* structure. Pass 
a record of type **PrintControlAttrs** containing the desired values.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*attributes* - The new PrintControlAttrs.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_ATTRS
    PrintControlAttrs   MSG_PRINT_CONTROL_GET_ATTRS();

Use this message to retrieve the values stored in the *PCI_attrs* structure. It 
will return a record of type **PrintControlAttrs** containing the desired 
values.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** The present print control attributes.

**Interception:** Unlikely.

### 17.4.3 Page Range Information
    PCI_startPage, PCI_endPage, PCI_startUserPage, 
    PCI_endUserPage, MSG_PRINT_CONTROL_SET_TOTAL_PAGE_RANGE, 
    MSG_PRINT_CONTROL_GET_TOTAL_PAGE_RANGE, 
    MSG_PRINT_CONTROL_SET_SELECTED_PAGE_RANGE, 
    MSG_PRINT_CONTROL_GET_SELECTED_PAGE_RANGE

The *PCI_startPage* and *PCI_endPage* fields should contain the page numbers 
of the first and last pages possible to print, known as the total page range. 
The *PCI_startUserPage* and *PCI_endUserPage* contain the beginning and 
ending page numbers of the range the user wants to print, often called the 
user page range. 

For documents whose page length may change (such as word processor 
documents), the page ranges should be updated whenever the Print dialog 
box goes up, at which time the application will be receiving a 
MSG_PRINT_NOTIFY_PRINT_DB. Normally, the user page range will start out 
being the same as the total page range. For most documents the first page of 
the total page range is page one, but applications which will include cover 
sheets might want the cover sheet to be page zero.

----------
#### MSG_PRINT_CONTROL_SET_TOTAL_PAGE_RANGE
    void    MSG_PRINT_CONTROL_SET_TOTAL_PAGE_RANGE(
            int     firstPage,
            int     lastPage);

The application should send this message to set the first and last page 
numbers of the document. You might want to send this message every time 
the document length changes or just every time the Print dialog box is put 
up. The Print Control warns the *PCI_output* object of the occurrence of the 
latter event with a MSG_PRINT_NOTIFY_PRINT_DB. 

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*firstPage* - The page number of the document's first page.

*lastPage* - The page number of the document's last page.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_TOTAL_PAGE_RANGE
    dword   MSG_PRINT_CONTROL_GET_TOTAL_PAGE_RANGE();

This message returns two integers. These integers are the numbers of the 
first and last pages of the range of possible pages. These values aren't 
necessarily the first and last pages of the range the user wants to print. Use 
MSG_PRINT_CONTROL_GET_SELECTED_PAGE_RANGE for that information.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** A double word. The high word is the first page; the low word is the last 
page.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_SET_SELECTED_PAGE_RANGE
    void    MSG_PRINT_CONTROL_SET_SELECTED_PAGE_RANGE(
            int     firstPage,
            int     lastPage);

This message takes the passed values and uses them as the first and last 
pages that the user wants to print.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*firstPage* - The page number of the first page to print.

*lastPage* - The page number of the last page to print.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_SELECTED_PAGE_RANGE
    dword   MSG_PRINT_CONTROL_GET_SELECTED_PAGE_RANGE();

This message returns the user's selected range of pages to print. 

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** A double word. The high word is the first page which the user wishes 
to print; the low word is the last page of this range.

**Interception:** Unlikely.

### 17.4.4 Document Size
    PCI_docSizeInfo, MSG_PRINT_CONTROL_SET_DOC_SIZE, 
    MSG_PRINT_CONTROL_GET_DOC_SIZE, 
    MSG_PRINT_CONTROL_SET_DOC_MARGINS, 
    MSG_PRINT_CONTROL_GET_DOC_MARGINS, 
    MSG_PRINT_CONTROL_SET_EXTENDED_DOC_SIZE, 
    MSG_PRINT_CONTROL_GET_EXTENDED_DOC_SIZE, 
    MSG_PRINT_CONTROL_SET_DOC_SIZE_INFO, 
    MSG_PRINT_CONTROL_GET_DOC_SIZE_INFO

It is possible to specify the size of the document when creating the Print 
Control. Note that this is the size of the document, not the size of the piece of 
paper. Ideally, the document should fit on the paper, though obviously in the 
case of huge documents like some spreadsheets, this may not be the case. The 
document size includes the margin size; it is possible to set the margin size 
as well.

The document size must be set correctly before the document is finished 
printing. If all documents the application produces have the same 
dimensions (or if you want some size to be the default), you may specify 
dimensions for the document when instantiating the print control. You may 
also set up document margins at this time.

The messages listed above get and set the document and margin sizes. You 
must use MSG_PRINT_CONTROL_GET_EXTENDED_DOC_SIZE and 
MSG_PRINT_CONTROL_SET_EXTENDED_DOC_SIZE when working with the 
dimensions of 32-bit extended documents.

----------
#### MSG_PRINT_CONTROL_SET_DOC_SIZE
    void    MSG_PRINT_CONTROL_SET_DOC_SIZE(
            int     width,
            int     height);

This message changes the values of the document size. It takes two integers, 
representing the new width and height for the document to use. It can only 
use 16-bit values, so use MSG_PRINT_CONTROL_SET_EXTENDED_DOC_SIZE 
when working with 32-bit extended graphics spaces.

Remember that the document size includes the document margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*width* - The document's width, in points.

*height* - The document's height, in points.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_DOC_SIZE
    dword   MSG_PRINT_CONTROL_GET_DOC_SIZE();

Use this message to retrieve the present document size. The size is returned 
as a width and height, each expressed in points. 

Note that if the size might be a 32-bit value (which might happen if the 
document uses an extended graphics space), you must use the 
MSG_PRINT_CONTROL_GET_EXTENDED_DOC_SIZE. If either dimension is a 
32-bit number, using a regular MSG_PRINT_CONTROL_GET_DOC_SIZE will 
result in an error.

Remember that the document size includes the document margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** A double word. The high word is the width, the low word is the height.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_SET_EXTENDED_DOC_SIZE
    void    MSG_PRINT_CONTROL_SET_EXTENDED_DOC_SIZE(
            PCDocSizeParams     *ptr);

This message changes the values of the document size using the two passed 
double integers as the new width and height to use. When working with 
normal 16 bit graphics spaces, use MSG_PRINT_CONTROL_SET_DOC_SIZE 
instead.

Remember that the document size includes the document margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to a **PCDocSizeParams** structure 
containing the document size.

**Return:** Nothing.

**Interception:** Unlikely.

**Structures:** The **PCDocSizeParams** structure has the following definition:

    typedef struct {
        dword       PCDSP_width;
        dword       PCDSP_height;
    } PCDocSizeParams;

----------
#### MSG_PRINT_CONTROL_GET_EXTENDED_DOC_SIZE
    void    MSG_PRINT_CONTROL_GET_EXTENDED_DOC_SIZE(
            PCDocSizeParams     *ptr);

Use this message to retrieve the present document size. It returns two double 
integers representing the width and height, expressed in points. If the size is 
a 16 bit value, you can use MSG_PRINT_CONTROL_GET_DOC_SIZE instead.

Remember that the document size includes the document margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to a **PCDocSizeParams** structure to hold 
document parameters.

**Return:** Nothing is returned explicitly.

*ptr* - The structure is filled with document size.

**Interception:** Unlikely.

**Structures:** The PCDocSizeParams structure has the following definition:

    typedef struct {
        dword       PCDSP_width;
        dword       PCDSP_height;
    } PCDocSizeParams;

----------
#### MSG_PRINT_CONTROL_SET_DOC_MARGINS
    void    MSG_PRINT_CONTROL_SET_DOC_MARGINS(
            PCMarginParams  *ptr);

Use this message to set new values for the document margins. It takes four 
arguments, the point values to use for the left, top, right, and bottom 
margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to a **PCMarginParams** structure with 
new document margins.

**Return:** Nothing.

**Interception:** Unlikely.

**Structures:** The **PCMarginParams** structure has the following definition:

    typedef struct {
        word        PCMP_left;
        word        PCMP_top;
        word        PCMP_right;
        word        PCMP_bottom;
    } PCMarginParams;

----------
#### MSG_PRINT_CONTROL_GET_DOC_MARGINS
    void    MSG_PRINT_CONTROL_GET_DOC_MARGINS(
            PCMarginParams  *ptr);

Use this message to get the present values for the document margins. It 
returns four integers. These integers represent the left, top, right, and 
bottom margins, expressed in typographer's points.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to a **PCMarginParams** structure which 
will hold return value.

**Return:** Nothing returned explicitly. 

*ptr* - Structure filled in with document margins.

**Interception:** Unlikely.

**Structures:** The **PCMarginParams** structure has the following definition:

    typedef struct {
        word        PCMP_left;
        word        PCMP_top;
        word        PCMP_right;
        word        PCMP_bottom;
    } PCMarginParams;

----------
#### MSG_PRINT_CONTROL_SET_DOC_SIZE_INFO
    void    MSG_PRINT_CONTROL_SET_DOC_SIZE_INFO(
            PageSizeReport  *ptr);

Use this message to set all of the information about the document size and 
orientation.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to a **PageSizeReport** structure.

**Return:** Nothing. 

**Interception:** Unlikely.

**Structures:** The **PageSizeReport** structure has the following definition:

    typedef struct {
        dword           PSR_width;
        dword           PSR_height;
        PageLayout      PSR_layout;
        PCMarginParams  PSR_margins;
    } PCMarginParams;

----------
#### MSG_PRINT_CONTROL_GET_DOC_SIZE_INFO
    void    MSG_PRINT_CONTROL_GET_DOC_SIZE_INFO(
            PageSizeReport  *ptr);

Use this message to set all of the information about the document size and 
orientation.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to an empty **PageSizeReport** structure 
which the message handler will fill in.

**Return:** Nothing. 

**Interception:** Unlikely.

**Structures:** The **PageSizeReport** structure has the following definition:

    typedef struct {
        dword           PSR_width;
        dword           PSR_height;
        PageLayout      PSR_layout;
        PCMarginParams  PSR_margins;
    } PCMarginParams;

### 17.4.5 Print Output Object
    PCI_output, MSG_PRINT_START_PRINTING, 
    MSG_PRINT_VERIFY_PRINT_REQUEST, MSG_PRINT_NOTIFY_PRINT_DB

Some object is going to describe the print jobs to the Print Control. When 
instantiating the Print Control, set the chosen object's name in the 
*PCI_output* field:

    PCI_output = <yourObject>;

There are three messages the Print Control may send to its output: 
MSG_PRINT_START_PRINTING, MSG_PRINT_VERIFY_PRINT_REQUEST, and 
MSG_PRINT_NOTIFY_PRINT_DB. The Print Output must respond to 
MSG_PRINT_START_PRINTING, which is the signal that it's time to describe 
a print job. The Print Output also must respond to 
MSG_PRINT_VERIFY_PRINTING if the PrintControl's PCA_VERIFY_PRINT bit 
has been set. Otherwise, it need not handle this message. Finally, the Print 
Output may have a handler for the MSG_PRINT_NOTIFY_DB, a message the 
PrintControl will send whenever the Print dialog box comes up or goes away.

#### 17.4.5.1 The Print Method
Assuming your geode doesn't try to do anything fancy with its own printing 
UI or weird scheduling, probably the most complicated thing you'll have to do 
when adding printing capability to your geode is write a Print Method.

This message must be handled by the Print Output object, though the 
handler probably won't be too complicated. In its simplest form, a 
MSG_PRINT_START_PRINTING handler could just call some graphics 
commands and finish off by sending 
MSG_PRINT_CONTROL_PRINTING_COMPLETED to the PrintControl.

The message is accompanied by a pointer back to the Print Control and a 
GString handle. Any graphics commands drawn to the GString handle will 
be retained and will become part of the print job. 

Before looking at examples, be warned that there are some requirements 
that every Print Method must meet:

+ Be sure the document size has been set correctly. Before the job is 
finished and you send MSG_PRINT_CONTROL_PRINTING_COMPLETED, 
you must be certain the document's size and margins have been set 
somewhere. This may have been set using the PCI_docSizeInfo field when 
defining the Print Control. If you haven't already let the Print Control 
know about the document size elsewhere, then you must send a 
MSG_SPOOL_PRINT_CONTROL_SET_DOC_SIZE, probably either in the 
MSG_PRINT_START_PRINTING or wherever your geode sets up document 
information in general.

+ Be certain the number of pages to be printed has been correctly set. If 
your application hasn't already set the page range (when the Print 
Control was created or when the Print dialog box goes up are popular 
times for this), send a MSG_PRINT_CONTROL_SET_TOTAL_PAGE_RANGE 
to your Print Control. Make sure you do it before sending in the 
MSG_PRINT_CONTROL_PRINTING_COMPLETED.

+ Be sure the document name has been set. If you use a 
GenDocumentGroup as your *PCI_docNameOutput*, this will be taken 
care of for you. Otherwise, some time before the 
MSG_PRINT_CONTROL_PRINTING_COMPLETED is sent, your 
*PCI_docNameOutput* object is going to be asked for the document name, 
and the job won't be spooled until your *PCI_docNameOutput* sends the 
correct name back to the Print Control.

+ The Print Method should end each page with a **GrNewPage()**; the last 
thing drawn before the print job ends is a **GrNewPage()**.

+ Finally, when the Print Method is finished describing the print job, it 
must end with a MSG_PRINT_CONTROL_PRINTING_COMPLETED. 
Otherwise, the Print Control has no way of knowing the job is ready.

When the Print Control receives the message 
MSG_PRINT_CONTROL_PRINTING_COMPLETED, it responds by cleaning up 
and sending the print job to the spooler.

Your handler can send a MSG_PRINT_CONTROL_REPORT_PROGRESS to the 
Print Control if it is spooling a large job and wants to reassure the user. If 
something goes wrong, or if MSG_PRINT_CONTROL_REPORT_PROGRESS 
returns a signal indicating the user wishes to cancel, 
MSG_PRINT_START_PRINTING should send a 
MSG_PRINT_CONTROL_PRINTING_CANCELLED instead.

----------
#### MSG_PRINT_START_PRINTING
    void    MSG_PRINT_START_PRINTING(
            optr            printControlOD,
            GStateHandle    gstate);

The handler for this message should call a number of graphics routines, 
using the passed GState. When done with graphics routines, the handler 
should send a MSG_PRINT_CONTROL_PRINTING_COMPLETED (or 
MSG_PRINT_CONTROL_PRINTING_CANCELED) to the Print Control. The 
handler may have to accomplish other tasks; see the above bulleted list.

**Source:** PrintControl object.

**Destination:** The object specified in PCI_output.

**Parameters:**  
*printControlOD* - The optr of the PrintControl object.

*gstate* - The GState handle to draw to.

**Return:** Nothing.

**Interception:** The Print output object must intercept this message, build the print job 
by drawing to the passed GState, then send 
MSG_PRINT_CONTROL_PRINTING_COMPLETED to the object in 
*printControlOD*.

#### 17.4.5.2 Verifying User Choices
You may write a handler for MSG_PRINT_VERIFY_PRINT_REQUEST, which 
will be sent to the Print Output if the PCA_VERIFY_PRINT bit has been set. 
The message will be sent after the user has dismissed the Print dialog box, 
after the system has had a chance to make sure the document fits on the 
paper but before the document is actually spooled.

This message gives the application a chance to check out the state of the UI 
and make sure that the user's choices are valid. Note that the PrintControl 
does its own checking to make sure that the document will fit on the page. 
After examining the UI, the handler must send back a 
MSG_PRINT_CONTROL_VERIFY_COMPLETED, passing the argument to say 
whether it's okay to print.

----------
#### MSG_PRINT_VERIFY_PRINT_REQUEST
    void    MSG_PRINT_VERIFY_PRINT_REQUEST(
            optr    printControlOD);

The handler for this message should make whatever checks are necessary to 
make sure that the user has made valid choices with the printing UI. The 
handler should then send back an indication of whether the user's choices are 
all right by means of a MSG_PRINT_CONTROL_VERIFY_COMPLETED.

**Source:** PrintControl object.

**Destination:** The object specified in *PCI_output*.

***Parameters:**  
printControlOD* - The optr of the PrintControl object.

**Return:** Nothing explicitly. However, the handler for this message must send a 
MSG_PRINT_CONTROL_VERIFY_COMPLETED back to the print 
control.

**Interception:** If you've set the PCA_VERIFY_PRINT flag, you must intercept this 
message correctly or else the print job will never start.

#### 17.4.5.3 Dialog Box Notification
The Print Control will send the Print Output a MSG_PRINT_NOTIFY_DB 
every time the Print dialog box comes up or goes away. Note that this 
message allows the application to update the page range and print group's UI 
at only those times that updates are needed.

This message will arrive with one argument, a pointer back to the 
PrintControl object that sent it. The handler for this message might wish to 
update the page range information using 
MSG_PRINT_CONTROL_SET_TOTAL_PAGE_RANGE and 
MSG_PRINT_CONTROL_SET_SELECTED_PAGE_RANGE.

----------
#### MSG_PRINT_NOTIFY_PRINT_DB
    void    MSG_PRINT_NOTIFY_PRINT_DB(
            optr                    printControlOD,
            PrintControlStatus      pcs);

The handler for this message can do any almost anything; this message 
signals that the Print dialog box has just come up or gone away.

**Source:** PrintControl object.

**Destination:** The object specified in *PCI_output*.

**Parameters:**  
*printControlOD* - The optr of the PrintControl object.

*pcs* - The status of the print control; either 
PCS_PRINT_BOX_VISIBLE or 
PCS_PRINT_BOX_NOT_VISIBLE.

**Return:** Nothing.

**Interception:** Some Print Output objects will intercept this message to set page 
ranges.

### 17.4.6 Document Name Output
    PCI_docNameOutput, MSG_PRINT_GET_DOC_NAME

The *PCI_docNameOutput* field must contain the optr of an object which can 
tell the Spooler the document's name. The document name is displayed on 
the Printer Control Panel and is how the user can figure out which job is 
which on the queue. If a GenDocumentGroup is set as the Print Control's 
*PCI_docNameOutput*, then that object will automatically do the right thing. 
If you use some other type of object, it must be prepared to supply document 
names on demand. This demand will come in the form of a 
MSG_PRINT_GET_DOC_NAME.

This message comes with a pointer back to the Print Control. The handler 
should respond by sending a MSG_PRINT_CONTROL_SET_DOC_NAME with a 
string containing the name.

----------
#### MSG_PRINT_GET_DOC_NAME
    void    MSG_PRINT_GET_DOC_NAME(
            optr    printControlOD);

If your *PCI_docNameOutput* object is not a GenDocumentGroup, then it must 
have a handler for this message.

**Source:** PrintControl object.

**Destination:** The object specified in PCI_docNameOutput.

**Parameters:**  
*printControlOD* - The optr of the PrintControl object.

**Return:** Nothing returned explicitly, but should send a 
MSG_PRINT_CONTROL_SET_DOC_NAME back to the Print Control.

**Interception:** If receiving object is a GenDocument, unlikely. Otherwise, interception 
is necessary.

### 17.4.7 The Default Printer
    PCI_defPrinter, MSG_PRINT_CONTROL_SET_DEFAULT_PRINTER, 
    MSG_PRINT_CONTROL_GET_DEFAULT_PRINTER

As this documentation has already stressed, one of the printing system's 
more important features is that it is device independent. However, certain 
special-purpose applications could conceivably depend on the user's printer 
type, and those applications may have use for the Print Control's 
*PCI_defPrinter* field. When the Print Control first appears, it will normally 
have the system default printer selected. Using the *PCI_defPrinter* option, 
another printer might be selected. The user may of course override the 
default printer and select another.

You can set or retrieve the value in *PCI_defPrinter* with the messages listed 
above. For information on them, see "Working with Instance Data" below.

There are utility routines to get information about the system default 
printer. There are commands to retrieve information about each of the 
installed printers so that the application may find out whether the desired 
type of printer is installed and, if so, what its printer number is. These 
routines are described in section 17.7.2.4 below.

### 17.4.8 Adding UI Gadgetry
    ATTR_PRINT_CONTROL_APP_UI

If the Print Control supplies UI that meets your geode's needs, feel free to 
skip this section. On the other hand, if you've decided you want a 
non-standard print trigger or want some specialized gadgetry to appear in 
the application's Print dialog box, it is possible to create objects and tell the 
Print Control to work with them. Make sure that any objects you want to use 
in this fashion aren't part of a generic or visual tree already. Also, these 
objects must be set not usable (~GS_USABLE).

Normally the user has two ways to bring up a Print dialog box: the User Print 
Trigger or the User Fax Trigger, both appearing in the File menu. You may 
specify a special trigger to be used in place of the standard trigger provided 
by the Print Control. Normally the default trigger is not only sufficient but 
preferable, if only because it will automatically display with the correct 
appearance for the specific UI. To define your own trigger object, create the 
object(s), setting the correct instance data. Then, when instantiating your 
Print Control, include an ATTR_GEN_CONTROL_APP_UI:

    ATTR_GEN_CONTROL_APP_UI = {yourTrigger};

When this option is used at all, normally the trigger object is a simple trigger. 
The action descriptor of this application-defined trigger should be

    GTI_output = <your_PrintControl>;
    GTI_method = MSG_SPOOL_PRINT_CONTROL_INITIATE_PRINT;

By working with the control's features, you may remove either or both of the 
default triggers.

The Print Control's vardata field ATTR_PRINT_CONTROL_APP_UI allows 
application-specific gadgetry to appear in the Print dialog box. Create the 
object(s) to be included and set their instance data, then use the 
ATTR_PRINT_CONTROL_APP_UI field to signal that the Print Control should 
include the gadgetry. 

    ATTR_PRINT_CONTROL_APP_UI = {yourPrintGroup};

When this option is used at all, usually a GenInteraction serves as the print 
group object, with children appropriate to the task at hand. This generic tree 
must not be destroyed unless the vardata field has been removed.

## 17.5 Print Control Messages
Now you have a rather good idea of what's required to set up a Print Control. 
Once it's been set up, your geode may send the following messages to it.

### 17.5.1 Common Response Messages
    MSG_PRINT_CONTROL_VERIFY_COMPLETED, 
    MSG_PRINT_CONTROL_SET_DOC_NAME

These messages are normally used only to respond to messages sent out by 
the Print Control.

----------
#### MSG_PRINT_CONTROL_VERIFY_COMPLETED
    void    MSG_PRINT_CONTROL_VERIFY_COMPLETED(
            Boolean     continue);

If you have decided to verify the user's Print dialog box choices, you must 
send this message back to the Print Control in response to 
MSG_PRINT_VERIFY_PRINT_REQUEST. If *true* is passed, the Print Control 
will continue preparing the job for printing. If the value passed is *false*, the 
print will be cancelled. It is up to the application to explain the problem to 
the user.

**Source:** Unrestricted, normally the PCI_output object.

**Destination:** Any PrintControl object.

**Parameters:**  
*continue* - Flag signalling whether printing should continue 
or be cancelled.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_SET_DOC_NAME
    void    MSG_PRINT_CONTROL_SET_DOC_NAME(
            char    * string);

This message must be sent by the *PCI_docNameOutput* before the print job 
can be spooled. It is sent in response to a MSG_PRINT_GET_DOC_NAME from 
the Print Control. The string argument should be something that will allow 
the user to recognize the print job in the Printer Control Panel.

**Source:** Unrestricted-normally the *PCI_docNameOutput* object.

**Destination:** Any PrintControl object.

**Parameters:**  
*string* - The document's name, as a null-terminated string. 
Maximum length of this string is 
FILE_LONGNAME_LENGTH+1.

**Return:** Nothing.

**Interception:** Unlikely.

### 17.5.2 Flow of Control Messages
    MSG_PRINT_CONTROL_PRINTING_COMPLETED, 
    MSG_PRINT_CONTROL_REPORT_PROGRESS, 
    MSG_PRINT_CONTROL_PRINTING_CANCELLED, 
    MSG_PRINT_CONTROL_REPORT_STRING, 
    MSG_PRINT_CONTROL_INITIATE_PRINT, MSG_PRINT_CONTROL_PRINT

Sending these messages to the Print Control signals that a new stage of 
printing is ready to begin.

----------
#### MSG_PRINT_CONTROL_PRINTING_COMPLETED
    void    MSG_PRINT_CONTROL_PRINTING_COMPLETED();

This message signals the Print Control that the application is finished 
describing the print job. The application must send this message (or a 
MSG_PRINT_CONTROL_PRINTING_CANCELLED) some time after receiving a 
MSG_PRINT_START_PRINTING.

**Source:** The object that received MGS_PRINT_START_PRINTING.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** Nothing.

**Interception:** Unlikely, unadvised.

----------
#### MSG_PRINT_CONTROL_REPORT_PROGRESS
    Boolean MSG_PRINT_CONTROL_REPORT_PROGRESS(
            PCProgressType  progress,
            int             pageOrPercent);

This message may be sent between the time the Print Output receives a 
MSG_PRINT_START_PRINTING and the time it sends a 
MSG_PRINT_CONTROL_PRINTING_COMPLETED. Its arguments include a 
progress type (page number or percentage) and a number representing the 
progress. It returns a Boolean value, which will normally be *true* but will be 
*false* if the user wishes to cancel printing.

The application should send out this message periodically over the course of 
a long print job, probably about once per page. Progress may be reported as 
a page number, a percentage of the job completed, or as a null-terminated 
string. 

If the return value is *false*, the application should stop drawing to the print 
GString immediately and send the Print Control a 
MSG_PRINT_CONTROL_PRINTING_CANCELLED. The return value may be 
safely ignored, but if the user wishes to cancel printing, it's polite to cancel 
immediately instead of making him wait until the 
MSG_PRINT_START_PRINTING has finished describing the entire job.

**Source:** The object that received MGS_PRINT_START_PRINTING when building 
a time-consuming job.

**Destination:** The PrintControl object specified by MSG_PRINT_START_PRINTING.

**Parameters:**  
*progress* - How progress is being reported, by page 
(PCPT_PAGE) or percent (PCPT_PERCENT).

*pageOrPercent* - How much progress has been made.

**Return:** Flag signalling that printing should continue. If the flag is *false* (i.e. 
zero), then the user wants to cancel printing, and it would be polite to 
stop.

**Interception:** Unlikely.

**Structures:** The *PCProgressType* enumeration is defined:

    typedef enum {
        PCPT_PAGE,
         PCPT_UNUSED1, /* Unused type */
         PCPT_PERCENT,
         PCPT_UNUSED2, /* Unused type */
         PCPT_TEXT
    } PCProgressType;

----------
#### MSG_PRINT_CONTROL_REPORT_PROGRESS_STRING
    @alias(MSG_PRINT_CONTROL_REPORT_PROGRESS) \ 
    Boolean MSG_PRINT_CONTROL_REPORT_PROGRESS_STRING(
            PCProgressType  progress,
            Chars           *progressString);

This message may be sent between the time the Print Output receives a 
MSG_PRINT_START_PRINTING and the time it sends a 
MSG_PRINT_CONTROL_PRINTING_COMPLETED. Its arguments include a 
text string giving some indication of progress. This message returns a 
Boolean value, which will normally be *true* but will be *false* if the user wishes 
to cancel printing.

The application should send out this message periodically over the course of 
a long print job, probably about once per page. Progress may be reported as 
a page number, a percentage of the job completed, or as a null-terminated 
string. 

If the return value is *false*, the application should stop drawing to the print 
GString immediately and send the Print Control a 
MSG_PRINT_CONTROL_PRINTING_CANCELLED. The return value may be 
safely ignored, but if the user wishes to cancel printing, it's polite to cancel 
immediately instead of making him wait until the 
MSG_PRINT_START_PRINTING has finished describing the entire job.

**Source:** The object that received MGS_PRINT_START_PRINTING when building 
a time-consuming job.

**Destination:** The PrintControl object specified by MSG_PRINT_START_PRINTING.

**Parameters:**  
*progress* - This must be PCPT_TEXT. The reason that this 
parameter has only one possible value is that this 
message is actually an alias of 
MSG_PRINT_CONTROL_REPORT_PROGRESS, 
handled by the same assembly routine.

*progressString* - A string of text, giving an indication of progress.

**Return:** Flag signalling that printing should continue. If the flag is *false* (i.e. 
zero), then the user wants to cancel printing, and it would be polite to 
stop.

**Interception:** Unlikely.

**Structures:** The *PCProgressType* enumeration is defined:

    typedef enum {
         PCPT_PAGE,
         PCPT_UNUSED1, /* Unused type */
         PCPT_PERCENT,
         PCPT_UNUSED2, /* Unused type */
         PCPT_TEXT
    } PCProgressType;

----------
#### MSG_PRINT_CONTROL_PRINTING_CANCELLED
    void    MSG_PRINT_CONTROL_PRINTING_CANCELLED();

The application may send this message to the PrintControl after receiving a 
MSG_PRINT_START_PRINTING. Do not send both this message and a 
MSG_PRINT_CONTROL_PRINTING_COMPLETED for a single print job.

The application may send this message to cancel a document while 
describing a print job, before it would send the 
MSG_PRINT_CONTROL_PRINTING_COMPLETED.

**Source:** Unrestricted, probably the *PCI_output*. 

**Destination:** The PrintControl object specified by MSG_PRINT_START_PRINTING.

**Parameters:** None.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_INITIATE_PRINT
    void    MSG_PRINT_CONTROL_INITIATE_PRINT();

This message, normally sent by the Print trigger in the File menu, is the 
signal that the Print Control should display the Print dialog box. Geodes with 
custom print triggers should make sure that those triggers send this message 
to the Print Control. Geodes using the Print Control's provided print trigger 
need not send this message.

**Source:** Unrestricted-typically the user print trigger (or fax trigger).

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_PRINT
    void    MSG_PRINT_CONTROL_PRINT();

This message, normally sent by the Print trigger in the Print dialog box, is 
the signal that the user has made his printing choices and is ready for the 
Print Control to verify those choices and spool the job. 

**Source:** Unrestricted-typically the Print trigger in the Print Dialog Box.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** Nothing.

**Interception:** Unlikely.

### 17.5.3 Working with Instance Data
    MSG_PRINT_CONTROL_SET_OUTPUT, 
    MSG_PRINT_CONTROL_GET_OUTPUT, 
    MSG_PRINT_CONTROL_SET_DOC_NAME_OUTPUT, 
    MSG_PRINT_CONTROL_GET_DOC_NAME_OUTPUT, 
    MSG_PRINT_CONTROL_SET_DEFAULT_PRINTER, 
    MSG_PRINT_CONTROL_GET_DEFAULT_PRINTER, 
    MSG_PRINT_CONTROL_GET_PRINT_MODE, 
    MSG_PRINT_CONTROL_GET_PAPER_SIZE, 
    MSG_PRINT_CONTROL_GET_PRINTER_MARGINS, 
    MSG_PRINT_CONTROL_CALC_DOCUMENT_DIMENSIONS, 
    MSG_PRINT_CONTROL_VERIFY_DOC_MARGINS, 
    MSG_PRINT_CONTROL_VERIFY_DOC_SIZE

The Print Control handles many messages which allow the geode to retrieve 
and alter the values of the instance data.

----------
#### MSG_PRINT_CONTROL_SET_OUTPUT
    void    MSG_PRINT_CONTROL_SET_OUTPUT(
            optr    objectPtr);

This message, used very rarely, specifies that a different object should 
become the Print Output. Pass the optr of the object to use.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*objectPtr* - The optr of the new *PCI_output*.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_OUTPUT
    optr    MSG_PRINT_CONTROL_GET_OUTPUT();

This message returns the optr of the Print Output.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** The optr of the current *PCI_output* object.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_SET_DOC_NAME_OUTPUT
    void    MSG_PRINT_CONTROL_SET_DOC_NAME_OUTPUT(
            optr    document);

This message, used very rarely, specifies that a different object should 
become the Document Name Output object. Pass the optr of the object to use.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*objectPtr* - The optr of the new *PCI_docNameOutput*.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_DOC_NAME_OUTPUT
    optr    MSG_PRINT_CONTROL_GET_DOC_NAME_OUTPUT();

This message returns the optr of the Document Name Output object.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** The optr of the Document Name output object.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_SET_DEFAULT_PRINTER
    void    MSG_PRINT_CONTROL_SET_DEFAULT_PRINTER(
            int     printerNum);        /* -1 for system's default printer */

This message sets the application default printer, to be used the next time 
the print dialog box appears. Pass the printer number of the printer to use. 
Passing a value of -1 means that the system default printer should be used.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*printerNum* - Number of the new default printer.

**Return:** Nothing.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_DEFAULT_PRINTER
    int     MSG_PRINT_CONTROL_GET_DEFAULT_PRINTER();

Use this message to retrieve the number of the present application default 
printer. You may not assume that this printer is the one the user is printing 
on, since the user may have changed to another printer. A return value of -1 
means that the application will use the system default printer.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

**Return:** The number of the default printer.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_PRINT_MODE
    byte    MSG_PRINT_CONTROL_GET_PRINT_MODE(); 

This message retrieves the current user-selected print mode, if any. Its 
possible return values include zero, meaning no mode has been selected; 
PM_GRAPHICS_LOW_RES; PM_GRAPHICS_MED_RES; 
PM_GRAPHICS_HI_RES; PM_TEXT_DRAFT; and PM_TEXT_NLQ.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:** None.

Return: A **PrinterOutputModes** signalling the printer's quality.

**Interception:** Unlikely.

**Structures:** The **PrinterOutputModes** structure has the following definition:

    typedef ByteFlags PrinterOutputModes;
    #define POM_GRAPHICS_LOW 0x10 
    #define POM_GRAPHICS_MEDIUM 0x08 
    #define POM_GRAPHICS_HIGH 0x04
    #define POM_TEXT_DRAFT 0x02 
    #define POM_TEXT_NLQ 0x01

Each flag indicates the specified quality is available. Two useful masks which 
have been set up are PRINT_GRAPHICS and PRINT_TEXT.

----------
#### MSG_PRINT_CONTROL_GET_PAPER_SIZE
    void    MSG_PRINT_CONTROL_GET_PAPER_SIZE(
            PCMargineParams         *retVal); 

Use this message to retrieve the Print Control's present paper size.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*retVal* - An empty **PCMarginParams** structure which the 
message handler will fill.

**Return:** Nothing returned explicitly; the *retVal* structure will be filled.

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_GET_PAPER_SIZE_INFO
    void    MSG_PRINT_CONTROL_GET_DOC_SIZE_INFO(
            PageSizeReport  *ptr);

Use this message to set all of the information about the document size and 
orientation.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*ptr* - Pointer to an empty **PageSizeReport** structure 
which the message handler will fill in.

**Return:** Nothing. 

**Interception:** Unlikely.

**Structures:** The **PageSizeReport** structure has the following definition:

    typedef struct {
        dword           PSR_width;
        dword           PSR_height;
        PageLayout          PSR_layout;
        PCMarginParams          PSR_margins;
    } PCMarginParams;

----------
#### MSG_PRINT_CONTROL_GET_PRINTER_MARGINS
    void    MSG_PRINT_CONTROL_GET_PRINTER_MARGINS(
            MarginDimensions        *retVal,
            Boolean         setMargins); 

This message returns the margins enforced by the requested printer. If the 
boolean argument is *true*, the document margins will be set to be the same as 
the printer margins.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*setMargins* - Pass TRUE to set the document's margins equal to 
the printer's margins, FALSE to leave the margins 
as they are.

*retVal* - A pointer to an empty **MarginDimensions** 
structure to be filled in by the handler.

**Return:** The printer's enforced margins.

**Structures:** The following structure keeps track of all four of a document's margins.

    typedef struct {
        int     leftMargin;     /* measured in points */
        int     topMargin;
        int     rightMargin;
        int     bottomMargin;
    } MarginDimensions;

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_CALC_DOC_DIMENSIONS
    void    MSG_PRINT_CONTROL_CALC_DOCUMENT_DIMENSIONS(
            PageSizeReport  *ptr); 

This message calculates the maximum printable area allowed by the 
user-selected paper size and printer-mandated margins. This message 
automatically resets the document size and margins to hold these values.

Note that this message correctly handles rotated pages.

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*retValue* - A pointer to a **DocumentSize** structure to hold 
return value.

**Return:** Nothing returned explicitly.

*retValue* - The pointer to the filled **DocumentSize** structure.

**Structures:** The following structure holds a document's size, and the margin 
information used to place the top left point.

    typedef struct {
        int     leftMargin;     /* measured in points */
        int     topMargin;
        int     width;
        int     height;
    } DocumentSize;

**Interception:** Unlikely.

----------
#### MSG_PRINT_CONTROL_CHECK_IF_DOC_WILL_FIT
    Boolean     MSG_PRINT_CONTROL_CHECK_IF_DOC_WILL_FIT(
                Boolean     warning); 

This message returns true if the passed document margins will fit on the 
selected printer. 

**Source:** Unrestricted.

**Destination:** Any PrintControl object.

**Parameters:**  
*warning* - If set, the PrintControl will deliver a warning to the 
user.

**Return:** Indication of whether the printer can work with the current margins 
(*true* if so, *false* if not).

**Interception:** Unlikely.

## 17.6 Page Size Control
    MSG_PRINT_REPORT_PAGE_SIZE, SpoolSetDocSize(), 
    MSG_PZC_GET_PAGE_SIZE, MSG_PZC_SET_PAGE_SIZE, 


Thanks to the Spooler, applications don't need to worry about what page size 
the user is working with. If a document is bigger than a printer's paper, then 
the Spooler will automatically tile the job onto as may sheets of paper as 
necessary. However, some applications will need to make the user decide on 
a page size. Page layout programs need to know what size page the user is 
working with. Spreadsheet programs need to make sure that cells aren't split 
in half by a page break.

Applications which ask the user to select a page size should incorporate a 
PageSizeControl into their generic tree. This generic control object provides 
a dialog box containing page setup choices for the user.

This dialog box contains UI allowing the user to specify what sort of paper the 
document is to be printed to: envelope, labels, or regular paper. The user may 
then specify a set of dimensions within that type; paper would have choices 
including letter sized, legal sized, and A4. The user can even set up a set of 
custom page dimensions.

Most applications which include a PageSizeControl will probably keep track 
of their own page size. In this case, use **SpoolSetDocSize()** to update the 
PageSizeControl's UI when changing the page size. When the user changes 
the paper size in the PageSizeControl, the control's output will receive a 
MSG_PRINT_REPORT_PAGE_SIZE.

Applications which do not keep track of the page size but want to include a 
PageSizeControl may do so. However, these applications will probably need 
to find out the page size at a specific time, not just receive notification every 
time the user changes the page size. This doesn't really fit the normal 
controller model, and in fact such applications are asking the 
PageSizeControl to act not like a controller, but like an ordinary piece of UI 
gadgetry. Setting the PZCA_ACT_LIKE_GADGET flag in the PageSizeControl's 
*PZCI_attrs* instance field will make the PageSizeControl act like a gadget.

If the PageSizeControl is to act as a controller, it must appear on the 
GenApplication's GAGCNLT_SELF_LOAD_OPTIONS GCN list, as does the 
PrintControl.

No matter how an application interacts with its PageSizeControl, it will work 
with the **PageSizeReport** structure.

    typedef struct {
        dword               PSR_width;
        dword               PSR_height;
        PageLayout          PSR_layout;
        PCMarginParams      PSR_margins;
    } PageSizeReport;

    typedef WordFlags PageLayout;
    typedef enum {
        PT_PAPER,
        PT_UNUSED1, /* Unused type */
        PT_ENVELOPE,
        PT_UNUSED2, /* Unused type */
        PT_LABEL
    } PageType;

    /* Specifying a PageLayout is accomplished by 
     * OR-ing together the parts of the layout.
     * Exactly what those parts are depends on the 
     * PageType: */

    /* Paper Layouts: 
     *  (PT_PAPER | (PO_orient << 3) ) */
    typedef ByteEnum                PaperOrientation;
    #define PO_PORTRAIT                 0x00
    #define PO_LANDSCAPE                0x01

    typedef WordFlags               PageLayoutPaper;
    #define PLP_ORIENTATION             0x0008
    #define PLP_TYPE                    0x0004 /* PT_PAPER */

    /* Envelope Layouts: 
     * (PT_ENVELOPE | (EP_path << 5) | (EO_ori << 3)) */
    typedef ByteEnum                EnvelopePath;
    #define EP_LEFT                     0x00
    #define EP_CENTER                   0x01
    #define EP_RIGHT                    0x02

    typedef ByteEnum                EnvelopeOrientation;
    #define EO_PORTAIT_LEFT             0x00
    #define EO_PORTAIT_RIGHT            0x01
    #define EO_LANDSCAPE_UP             0x02
    #define EO_LANDSCAPE_DOWN           0x03

    typedef WordFlags               PageLayoutEnvelope;
    #define PLE_PATH                    0x0040
    #define PLE_ORIENTATION             0x0010
    #define PLE_TYPE                    0x0004 /*PT_ENVELOPE*/

    /* Label Layouts:
     *  (PT_LABEL | (rows <<8) | (cols << 3)) */
    typedef WordFlags               PageLayoutLabel;
    #define PLL_ROWS                    0x7e00 /* # rows */
    #define PLL_COLUMNS                 0x01f8 /* # cols */
    #define PLL_TYPE                    0x0004 /* PT_LABEL */

----------
**Code Display 17-3 PageSizeControl Features**

    typedef ByteFlags   PageSizeControlFeatures;
    /* The following flags may be combined with | and &:
        PSIZECF_MARGINS,            ( Set margin sizes )
        PSIZECF_CUSTOM_SIZE,        ( Custom dimension ranges )
        PSIZECF_LAYOUT,
        PSIZECF_SIZE_LIST,
        PSIZECF_PAGE_TYPE           ( Choice of standard dimensions ) */

    #define PSIZEC_DEFAULT_FEATURES (PSIZECF_PAGE_TYPE | PSIZECF_SIZE_LIST | \
                                    PSIZECF_LAYOUT | PSIZECF_CUSTOM_SIZE) 

----------

----------
#### MSG_PRINT_REPORT_PAGE_SIZE
    void    MSG_PRINT_REPORT_PAGE_SIZE(
            PageSizeReport *psr);

The page size control's output object should have a handler for this message. 
The control will send this message whenever the user has changed the page 
size. The handler may wish to change the bounds of some appropriate object, 
store the new bounds somewhere, or take some other action.

**Source:** PageSizeControl object.

**Destination:** The *GCI_output* object.

**Parameters:**  
*psr* - A pointer to a **PageSizeReport** structure 
containing the page type and size.

**Return:** Nothing.

**Interception:** The handler for this message should store the page size information. It 
may set the bounds of one or more visual objects.

In addition to the standard set of generic Control instance data, the Page Size 
control includes some of its own. This data, as you might expect, is largely 
concerned with paper size and setup. The fields are listed in Code 
Display 17-4.
Only work with instance data if the PageSizeControl is in 
Gadget mode.

**PageSizeControls** use the **PageLayout** data structure to hold and pass 
layout information such as page orientation. One part of this structure gives 
the type of page being described: paper, envelope, or label. The other parts of 
the structure have different meanings depending on the type of page. For 
paper, the layout keeps track of the orientation. For envelopes, the structure 
contains both path and orientation information. For labels, this structure 
keeps track of how many labels are on the sheet in each direction; these 
numbers are bounded by MAXIMUM_LABELS_ACROSS and 
MAXIMUM_LABELS_DOWN.

----------
**Code Display 17-4 Page Size Control Instance Data**

    @instance PageSizeControlAttrs              PZCI_attrs = 0;
        /* Possible attributes:
         * PZCA_ACT_LIKE_GADGET
         * PZCA_PAPER_SIZE
         * PZCA_INITIALIZE              */

        /* Current page dimensions and type/layout */
    @instance dword             PZCI_width = 0;
    @instance dword             PZCI_height = 0;
    @instance PageLayout        PZCI_layout = 0;
    @instance PCMarginParams    PZCI_margins = { 0, 0, 0, 0 };

    /* The following vardata field is internal: */
    @vardata PageSizeControlMaxDimensions TEMP_PAGE_SIZE_CONTROL_MAX_DIMENSIONS;

    /* Its structure is defined: 
    typedef struct {
        dword PZCMD_width; ( maximum width )
        dword PZCMD_height; ( maximum height )
    } PageSizeControlMaxDimensions; */


    /* Attribute to allow applications to be made aware of every change made
     * to gadgetry in the PageSizeControl. This is especially useful for applications
     * that want to add UI that will be dependent upon the state of this
     * controller. No effort is made to eliminate the redundant output of data. */

    @vardata PageSizeControlChanges ATTR_PAGE_SIZE_CONTROL_UI_CHANGES;

    /* This field has structure:
        typedef struct {
         optr PSCC_destination; ( destination for message )
         Message PSCC_message; ( message to be sent )
        } PageSizeControlChanges; */

    The message should follow the prototype: */
    @prototype void PAGE_SIZE_UI_CHANGES_MSG(PageSizeReport _far *psr )

----------

*PZCI_width, PZCI_height*  
These integers represent the current page dimensions, 
measured in points. Note that these values may never go 
outside the bounds described by the constants 
MINIMUM_PAGE_WIDTH_VALUE, 
MINIMUM_PAGE_HEIGHT_VALUE, 
MAXIMUM_PAGE_WIDTH_VALUE, and 
MAXIMUM_PAGE_HEIGHT_VALUE.

PZCI_layout  
This field contains a **PageLayout** structure describing the 
current page layout settings.

PZCI_margins  
This field contains the page's margins.

The following messages allow applications to work with a PageSizeControl 
directly, like a regular UI gadget. There is one message which the 
PageSizeControl will send to its output, and several that any object may send 
to the controller.

Note that these messages are only useful if the PageSizeControl is operating 
in gadget mode (i.e. if its PZCA_ACT_LIKE_GADGET bit is set). Otherwise, the 
control's default behavior would override that requested by these messages. 

The **PageSizeControlAttrs** record has four flags:

PZCA_ACT_LIKE_GADGET  
The PageSizeControl object will act like any other generic 
gadget and will not respond to normal controller notifications.

PZCA_PAPER_SIZE  
The PageSizeControl object will display paper sizes instead of 
document sizes. Setting this would be very rare for 
applications.

PZCA_INITIALIZE  
The PageSizeControl object will be initialized to system default 
values if PZCA_ACT_LIKE_GADGET is also set.

PZCA_LOAD_SAVE_OPTIONS  
Allow the controller to load and save the user's options.

----------
#### MSG_PZC_GET_PAGE_SIZE
    void    MSG_PZC_GET_PAGE_SIZE(
            PageSizeReport      *psr);

This message returns the present page size in terms of width, height, and 
layout, stored in a **PageSizeReport**. Note that the PageSizeControl will be 
sending out a MSG_PRINT_REPORT_PAGE_SIZE every time the user applies 
changes to these values. 

Only send this message if the PageSizeControl is operating in gadget mode.

**Source:** Unrestricted, as long as the PageSizeControl is in gadget mode.

**Destination:** PageSizeControl object.

**Parameters:**  
*psr* - A pointer to a **PageSizeReport** structure in which 
the page size will be returned.

**Return:** The **PageSizeReport** structure pointed to by *psr* will be filled with the 
page information.

**Interception:** Unlikely.

----------
#### MSG_PZC_SET_PAGE_SIZE
    void    MSG_PZC_SET_PAGE_SIZE(
            PageSizeReport *    psr);       

This message changes the current page size. It takes a **PageSizeReport** 
data structure containing the new dimensions and layout. The Apply trigger 
of a Page Size Control normally sends this message, passing the present user 
page size as arguments.

Only send this message if the PageSizeControl is operating in gadget mode.

**Source:** Unrestricted, as long as the PageSizeControl is in gadget mode.

**Destination:** PageSizeControl object.

**Parameters:**  
*psr* - pointer to a **PageSizeReport** structure 
containing page information.

**Return:** Nothing.

**Interception:** Unlikely.

## 17.7 Other Printing Components
Most geodes can fulfill all their drawing needs by working with their Print 
Control. The system has two other main parts that work with printing: the 
first is the spool library, which controls scheduling; the second is a collection 
of printer drivers, which handle interactions between the printing devices 
and the rest of the system.

### 17.7.1 Spooler and Scheduling
    SpoolInfo(), SpoolHurryJob(), SpoolDelayJob(), 
    SpoolModifyPriority(), SpoolVerifyPrinterPort(), 
    SpoolCreateSpoolFile(), SpoolDelJob(), 
    MSG_PRINT_NOTIFY_PRINT_JOB_CREATED

Geodes may invoke certain routines that affect the Spooler, but for the most 
part they should avoid this practice. These routines allow the geode to take 
advantage of some of the Spooler's scheduling capabilities, so the geode can 
hurry some jobs and delay others. Most applications should not be concerned 
with the scheduling of printer jobs. If the user thinks that some job should be 
given a higher priority, he can work with the Printer Control Panel. Many of 
the following routines require the print job's ID. Applications that will use 
these routines should have a handler for 
MSG_PRINT_NOTIFY_PRINT_JOB_CREATED, which has reference material 
at the end of this section.

For those applications that will be scheduling, the **SpoolHurryJob()** and 
**SpoolDelayJob()** routines may come in handy. Passed a job ID, these 
routines move the associated job to the front or back of the queue (see 
Figure 17-9). **SpoolModifyPriority()** changes the priority of a spool thread 
so that it may print in the foreground or be moved further into the 
background. Each function returns a code describing the Spool's Operating 
Status, alerting the geode if the job wasn't found, the queue was empty, the 
port couldn't be verified, or even if the operation was successful.

![image info](Figures/Fig17-9.png)  
**Figure 17-9** *Hurrying and Delaying Jobs*  
*Print jobs that have already been sent to the printer are not shown here.*

The **SpoolInfo()** routine returns the list of jobs on a queue or information 
about any individual job. The other kind of information application writers 
might be interested in is the printer port status. It's sometimes unwise to 
spool a job when the printer is having problems, and 
**SpoolVerifyPrinterPort()** can give some warning about problems.

While the above routines are used only rarely in applications, there are 
others that are used even less often. **SpoolCreateSpoolFile()** creates a 
spool file, which may be used to create a pre-generated job for the printer 
kept handy. Such a file would only be useful to applications that can only 
print one thing and don't mind leaving around an extra spool file. 
**SpoolDelJob()** deletes jobs from the queue.

----------
#### MSG_PRINT_NOTIFY_PRINT_JOB_CREATED
    void    MSG_PRINT_NOTIFY_PRINT_JOB_CREATED(
            word jobID);        

The PrintControl sends this message when it has sent the job to the print 
queue. Applications may intercept this message to retrieve the new job's ID 
number or for any other reason.

**Source:** PrintControl object.

**Destination:** *GCI_output* object.

**Parameters:**  
*jobID* - Print job's ID.

**Return:** Nothing.

**Interception:** Applications that want to know the ID of created jobs should intercept 
this message. Note that this behavior is not encouraged, and most 
applications will want to leave meddling with a print job's status up to 
the user's discretion.

### 17.7.2 Printer Drivers
Printer Drivers are very important but at the same time not something that 
any geode you write is going to interact with. Printer Drivers take care of 
translating the device-independent graphics commands into forms that 
individual models of printers can work with. Drivers also provide the Spooler 
with information about the printer and provide printer-specific UI to the 
Print Control.

As long as a geode uses only device-independent graphics commands, that 
geode need not worry about printer drivers. In fact, the only geodes that 
require knowledge of printer drivers are those that use Raw Mode (see below) 
printing and access the printer drivers directly. To incorporate Raw Mode 
printing into a geode (normally a bad idea), keep reading. Whether or not the 
geode you're writing will interact with Printer Drivers, you might want to 
continue reading this section to find out what printer drivers do.

#### 17.7.2.1 Dumb Printers
When dealing with a printer that has no page description language, the 
Spooler and Printer Driver have to be clever. First the Spooler allocates an 
offscreen bitmap and plays back the contents of the spool file into the bitmap. 
It does this through the **vidmem** driver, which has the same interface as a 
video driver but draws to offscreen bitmaps instead of to a device. Once the 
bitmap is complete, the Spooler starts feeding the bitmap to the Printer 
Driver, which translates the bitmap into a series of commands to the printer. 
If (as is often the case) the printer doesn't have enough memory to absorb a 
whole page's worth of data at a time, the Printer Driver feeds in one 
horizontal band, or "swath" of the bitmap at a time. This also allows for better 
backgrounding: The driver can feed in a swath, then block and allow other 
threads to complete other tasks until the printer is ready for the next swath.

#### 17.7.2.2 Smart Printers
When dealing with an "intelligent" printer, a printer that has its own page 
description language, the Spooler bypasses most of the work it has to do 
when dealing with dumb printers. Since an intelligent printer presumably 
has some commands with GEOS equivalents, the Spooler doesn't bother with 
creating a bitmap but just passes the GString on to the Printer Driver. The 
Printer Driver then builds a page description in the printer's native language 
based on the contents of the GString.

#### 17.7.2.3 Raw Mode
If you want to use some printer-specific command that doesn't have an 
equivalent in the GEOS graphics system, you will use raw mode. Raw Mode 
printing expects that you know what sorts of commands the printer will be 
expecting. If you try to send PostScript commands to a printer that doesn't 
understand PostScript, probably the results won't be the desired output.

#### 17.7.2.4 Printer Information and Manipulation
    SpoolGetNumPrinters(), SpoolGetPrinterString(), 
    SpoolGetPrinterInfo(), SpoolCreatePrinter(), 
    SpoolDeletePrinter(), SpoolGetDefaultPrinter(), 
    SpoolSetDefaultPrinter()

If you want to find out something about a printer without accessing the 
printer driver directly, the spool library provides a number of utility routines 
that work with printer drivers.

Most of these functions specify which of a user's printers they wish to work 
with by passing its printer number. The first installed printer will be printer 
number zero, the second printer will be number one, and so on. Use the 
**SpoolGetNumPrinters()** routine to find out how many printers have been 
installed. Once you have this number, you have an upper bound for printer 
numbers to test. There will always be fewer than 
MAXIMUM_NUMBER_OF_PRINTERS printers.

The **SpoolGetPrinterString()** and **SpoolGetPrinterInfo()** routines 
return information about the printer associated with the passed printer 
number. **SpoolGetPrinterString()** returns the a string containing the 
name of the printer, such as "HP DeskJet on COM1." The printer string will 
be of length at most MAXIMUM_PRINTER_NAME_LENGTH. 
**SpoolGetPrinterInfo()** returns various pieces of information helpful to 
geodes that work with printers directly.

The **SpoolCreatePrinter()** and **SpoolDeletePrinter()** routines can install 
and uninstall printers, tasks probably best left to the Preferences Manager. 
The **SpoolDeletePrinter()** routine uninstalls the printer corresponding to 
the passed printer number. It is easy to use but not something that most 
geodes should be doing. **SpoolCreatePrinter()** installs a new printer, 
returning the new printer's printer number. Note that any geode calling this 
latter function is going to be expected to provide considerable information 
about the printer being installed. Again, this is not a routine that many 
geodes should be using.

**SpoolGetDefaultPrinter()** returns the number of the system-default 
printer. **SpoolSetDefaultPrinter()** makes the printer associated with the 
passed printer number the new system-default printer. Note that these 
routines are concerned with the system-default printer as opposed to the 
application-default printer. Most applications won't specify an application 
specific default printer. For these applications, the system-default printer 
will be the default. Of course, the user will still be allowed to select any 
printer, ignoring the default.

### 17.7.3 Page Size Related Routines
    SpoolGetNumPaperSizes(), SpoolGetPaperString(), 
    SpoolConvertPaperSize(), SpoolCreatePaperSize(), 
    SpoolDeletePaperSize(), SpoolGetPaperSizeOrder(), 
    SpoolSetPaperSizeOrder()

Most geodes won't be too concerned with what sort of page size choice the 
user has made. If the document fits on the page, all's well. If the document 
doesn't fit, the Spooler will tile the job using only as many pieces of paper as 
necessary. If the document size should be the same as the page size, the geode 
can send a MSG_PRINT_CONTROL_CALC_DOCUMENT_DIMENSIONS to the 
print control.

There are several routines which deal with page sizes. Most applications 
have no reason to call these functions, and we will not discuss them in detail 
here. Some general notes are in order for those programmers who might work 
with these functions, though.

It takes two indexes to access a page size. The first is the **PageType**: paper, 
envelope, or label. Within each type, there are up to MAX_PAPER_SIZES page 
sizes. To find out the number of page sizes presently defined for a given type, 
use the **SpoolGetNumPaperSizes()** routine. To find the number of a paper 
size associated with a **PageType** and a given set of dimensions, use the 
**SpoolConvertPaperSize()** routine. For the text string describing the size 
of the page, call **SpoolGetPaperString()** (the buffer to hold the string 
should be of size MAX_PAPER_STRING_LENGTH.

To find out the order in which the page sizes of a given type are stored, call 
**SpoolGetPaperSizeOrder()**. You may give a new order by calling 
**SpoolSetPaperSizeOrder()**. This will affect the order in which page sizes 
are presented in a page size control.

To create a new page size, call **SpoolCreatePaperSize()**. This routine takes 
a page type, a set of dimensions, and a string to describe those dimensions to 
the user. It returns a new page size number. To destroy a page size, call 
**SpoolDeletePaperSize()**. 

## 17.8 Debugging Tips
Though GEOS provides an easy-to-use printer interface, mistakes are sure to 
happen. You may find the following information useful in catching your 
mistakes or for testing rigorously.

Under Swat, you can see the various threads the Spooler works with. There 
is always one spool thread running in the background. When the first job is 
queued on a printer, you can see that a new thread is created for that printer. 
When the last job for a printer is completed, the thread exits again. 

Here are the most common printing mistakes:

+ Forgetting to add the Print Control object to the application's Active List 
(the MGCNLT_ACTIVE_LIST GCN list).

+ Not setting the document name (see section 17.4.6 above).

+ Setting the number of pages (page range) incorrectly.

[Impex Library](oimpex.md) <-- [Table of Contents](../objects.md) &nbsp;&nbsp; --> [Graphic Object Library](ogrobj.md)

