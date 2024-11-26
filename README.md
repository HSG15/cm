I have a requirement that .. After creating a sales order a mail should  be sent from one employee to another employee(Accounting Manager) having two button in it Approve and Reject.. If he clicks approve then the Sales order should be approved and if click reject then it will ask to enter a rejection reason and in that field of that SO rejection reason should get fetched. I will use user event after submit and  suitelet for it ..<br>below are my code.. if you want can modify<br>user event:<br>/**
 * @NApiVersion 2.1
 * @NScriptType UserEventScript
 */
define(['N/record', 'N/log', 'N/email', 'N/runtime'],
    /**
 * @param{record} record
 */
    (record, log, email, runtime) => {
        /**
         * Defines the function definition that is executed before record is loaded.
         * @param {Object} scriptContext
         * @param {Record} scriptContext.newRecord - New record
         * @param {string} scriptContext.type - Trigger type; use values from the context.UserEventType enum
         * @param {Form} scriptContext.form - Current form
         * @param {ServletRequest} scriptContext.request - HTTP request information sent from the browser for a client action only.
         * @since 2015.2
         */
        const beforeLoad = (scriptContext) => {
            log.debug('before load triggered')
        }

        /**
         * Defines the function definition that is executed before record is submitted.
         * @param {Object} scriptContext
         * @param {Record} scriptContext.newRecord - New record
         * @param {Record} scriptContext.oldRecord - Old record
         * @param {string} scriptContext.type - Trigger type; use values from the context.UserEventType enum
         * @since 2015.2
         */
        const beforeSubmit = (scriptContext) => {


        }

        /**
         * Defines the function definition that is executed after record is submitted.
         * @param {Object} scriptContext
         * @param {Record} scriptContext.newRecord - New record
         * @param {Record} scriptContext.oldRecord - Old record
         * @param {string} scriptContext.type - Trigger type; use values from the context.UserEventType enum
         * @since 2015.2
         */
        const afterSubmit = (scriptContext) => {
            log.debug('after submit triggered')

            var obj_newRecord = scriptContext.newRecord;
            log.debug('Record object is ', obj_newRecord)
            var billId = obj_newRecord.id;
            //var document = obj_newRecord.tranNumber
            log.debug('bill id: ' + billId)

            if (billId) {

                var vendorName = obj_newRecord.getValue({ fieldId: 'entity' }); 
                var billAmount = obj_newRecord.getValue({ fieldId: 'total' }); 
                var tranNumber = obj_newRecord.getValue({ fieldId: 'tranid' });
                log.debug('vendor name: ' + vendorName + ' & Bill Amount: ' + billAmount + ' & Tran No: ' + tranNumber)
                var currentUser = runtime.getCurrentUser().id
                var senderId = currentUser // Employee
                var recipientId = 1106 // Accounting Manager

                var subject = "Vendor Bill id:=" + tranNumber + " needs your approval";
                var body = "The following vendor bill has been ";
                body += "Vendor Name: " + vendorName + "\n";
                body += "Bill Amount: " + billAmount + "\n";
                body += "Bill ID: " + billId + "\n";

                const htmlContent = `<!DOCTYPE html>
                <html>
                <head>
                    <title>Vendor Bill Approval</title>
                </head>
                <body>
                    <p>Hello '`+ vendorName + `',</p>
                    <p>A new vendor bill has been submitted and requires your approval.</p>
                    <p>Bill Details:</p>

                    /*Approve Button*/
                    <a href="https://td2969844.app.netsuite.com/app/site/hosting/scriptlet.nl?script=966&amp;deploy=1&custpage_billid=`+ billId + `&custpage_approve=T" style="background-color: #4CAF50; /* Green */
                        border: none;
                        color: white;
                        padding: 15px 32px;
                        text-align: center;
                        text-decoration: none;
                        display: inline-block;
                        font-size: 16px;
                        margin: 4px 2px;
                        cursor: pointer;">Approve</a>

                    /*Reject Button*/
                    <a href="https://td2969844.app.netsuite.com/app/site/hosting/scriptlet.nl?script=966&amp;deploy=1&custpage_billid=`+ billId + `" style="background-color:  #f44336; /* Red */
                        border: none;
                        color: white;
                        padding: 15px 32px;
                        text-align: center;
                        text-decoration: none;
                        display: inline-block;
                        font-size: 16px;
                        margin: 4px 2px;
                        <div id="rejectionReasonContainer"></div> <!-- Container for rejection reason textarea -->Reject</a>

                    <p>Thank you.</p>
                </body>
                </html>`;

                try {
                    log.debug('mail send init ')
                    email.send({
                        author: senderId,
                        recipients: 'harishankar.giri@theblueflamelabs.com', //1106
                        subject: subject,
                        body: htmlContent
                    });
                    log.debug('mail sent')
                } catch (e) {
                    log.debug('error is ', e)
                }

            }
        }

        return { beforeLoad, beforeSubmit, afterSubmit }

    });
<br><br>suitelet:<br>/**
 * @NApiVersion 2.x
 * @NScriptType Suitelet
 */
define(['N/record', 'N/ui/serverWidget', , 'N/redirect'],

    (record, serverWidget, redirect) => {

        const onRequest = (scriptContext) => {
            if (scriptContext.request.method === 'GET') {
                var billId = scriptContext.request.parameters.custpage_billid;

                var form = serverWidget.createForm({
                    title: 'Enter Rejection Reason'
                });

                form.addField({
                    id: 'rejection_reason',
                    type: serverWidget.FieldType.TEXT,
                    label: 'Rejection Reason'
                });

                form.addSubmitButton({
                    label: 'Submit'
                });

                scriptContext.response.writePage(form);
            } else {
                var rejectionReason = scriptContext.request.parameters.rejection_reason;
                var billId = scriptContext.request.parameters.custpage_billid;

                // Update the record with rejection reason
                try {
                    const recordObj = record.load({
                        type: record.Type.SALES_ORDER,
                        id: billId
                    });
                    recordObj.setValue({
                        fieldId: 'custbody1',   
                        value: rejectionReason
                    });
                    recordObj.save();
                } catch (e) {
                    scriptContext.response.write('Error: ' + e.message);
                }

                // Send email notification or perform any other actions
                scriptContext.response.write('Rejection reason submitted successfully.');
            }
        }

        return {
            onRequest: onRequest
        };
    });

<br>
