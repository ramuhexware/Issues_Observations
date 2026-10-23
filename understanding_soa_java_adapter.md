# SOA Java Adapter Test Project Analysis

## Project Overview & Purpose

The **`soa-java-adapter-test`** project is a synthetic test fixture designed to validate static code parsers (such as CodeScout / Parser 3.0) and knowledge-graph extractors (Neo4j). Its primary purpose is to model how Oracle SOA Suite composites invoke Java code via:
1. **BPEL Java Embeddings (`<bpelx:exec language="java">`)**: Directly instantiating Java classes bundled in `SCA-INF/lib/*.jar`.
2. **EJB References (`<binding.ejb>`)**: Remote stateless session beans over WebLogic T3 protocol.
3. **Spring SCA Components (`<implementation.spring>`)**: Declarative Spring bean integrations in SCA.

---

## 1. Functional Flow

The loan application processing lifecycle follows this end-to-end execution path:

```
[JavaAdapterClient] (JAX-WS Client)
        │
        ▼ (SOAP / HTTP)
[JavaAdapterProcess.wsdl] -> javaadapter_client_ep
        │
        ▼ (SCA wire)
[composite.xml] -> LoanProcessOrchestration Component
        │
        ▼ (implementation.bpel)
[LoanProcessOrchestration.bpel]
        ├── 1. <bpelx:exec> -> DTICalculator.java (Calculates DTI & LTV)
        ├── 1b. <invoke>   -> CreditCheckEJB.java via CreditCheckPartner (Checks Credit)
        ├── 2. <bpelx:exec> -> OCRProcessor.java (Extracts & Classifies Documents)
        ├── 3. <bpelx:exec> -> DecisionEngine.java (Underwrites Loan Decision)
        ├── 3b. <invoke>   -> ComplianceCheckEJB.java via ComplianceCheckPartner (Validates Regulations)
        └── 4. <bpelx:exec> -> AmortizationCalculator.java (Calculates Rate & Payments)
        │
        ▼
Reply LoanProcessResponse to Client
```

### Detailed Step-by-Step Flow:
1. **Client Request**:
   [JavaAdapterClient.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-client/src/com/soa/client/JavaAdapterClient.java#L50-L60) sends a SOAP request (`processLoan`) to the endpoint `http://localhost:8080/soap/JavaAdapterProcess` defined in [JavaAdapterProcess.wsdl](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.wsdl#L50-L54).
2. **Composite Ingress**:
   [composite.xml](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/composite.xml#L30-L59) routes the incoming call from `javaadapter_client_ep` to the `LoanProcessOrchestration` component.
3. **BPEL Orchestration** ([LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel)):
   - **`ReceiveRequest` (Line 60)**: Receives `inputVariable` containing applicant income, debt, loan amount, and property value.
   - **Step 1 (`CalculateDTI`, Lines 71-96)**: Inline Java instantiates [DTICalculator](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/DTICalculator.java#L27-L49), computes `dtiRatio` and `ltvRatio`, and writes them back into BPEL variables.
   - **Step 1b (`InvokeCreditCheck`, Lines 118-129)**: Invokes the EJB reference `CreditCheckPartner` ([CreditCheckEJB.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/CreditCheckEJB.java#L29-L46)) to fetch applicant credit score.
   - **Step 2 (`ProcessOCR`, Lines 136-164)**: Inline Java instantiates [OCRProcessor](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/OCRProcessor.java#L27-L38) to parse and classify attached base64 documents (e.g., W-2, paystub).
   - **Step 3 (`RunDecisionEngine`, Lines 171-202)**: Inline Java instantiates [DecisionEngine](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/DecisionEngine.java#L31-L45) to determine approval status (`APPROVE`, `REFER`, `DECLINE`) and calculate `riskScore`.
   - **Step 3b (`InvokeComplianceCheck`, Lines 224-235)**: Invokes the EJB reference `ComplianceCheckPartner` ([ComplianceCheckEJB.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/ComplianceCheckEJB.java#L40-L80)) to test against Qualified Mortgage (QM) and high-cost loan thresholds.
   - **Step 4 (`CalculatePricing`, Lines 242-270)**: Inline Java instantiates [AmortizationCalculator](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/AmortizationCalculator.java#L33-L75) to calculate final interest rate and monthly amortization payment.
   - **`AssignResponse` & `ReplyResponse` (Lines 273-313)**: Maps computed values into `outputVariable` and returns the `LoanProcessResponse` payload to the client.

---

## 2. Issues, Flaws, and Inconsistencies in the Application

If this project were deployed to an Oracle SOA Suite server or executed against strict static analysis/validation tools, multiple fatal issues and functional defects would occur:

### A. Critical BPEL & WSDL Specification Issues
1. **Missing WSDL & Schema `<import>` in BPEL**:
   - In [LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel), there is **no `<import>` statement** importing [JavaAdapterProcess.wsdl](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.wsdl).
   - In standard WS-BPEL 2.0, variables referencing message types outside the BPEL document cannot resolve without an `<import namespace="..." location="..." importType="http://schemas.xmlsoap.org/wsdl/"/>`.
2. **Namespace Mismatch for Input/Output Messages**:
   - [LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L37-L38) declares:
     ```xml
     <variable name="inputVariable" messageType="tns:LoanProcessRequestMessage"/>
     ```
     where `tns` is bound to `http://xmlns.oracle.com/JavaAdapterTest/LoanProcessOrchestration`.
   - However, `LoanProcessRequestMessage` is defined in [JavaAdapterProcess.wsdl](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.wsdl#L3) under target namespace `http://xmlns.oracle.com/JavaAdapterTest/JavaAdapterProcess`.
3. **Missing Partner Link Types (`<plnk:partnerLinkType>`)**:
   - BPEL declares partner links `tns:LoanProcessPLT`, `tns:CreditCheckPLT`, and `tns:ComplianceCheckPLT` ([lines 24-34](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L24-L34)).
   - **None of these partner link types exist in any WSDL in the project.**
4. **Missing EJB Message and Type Definitions**:
   - BPEL declares variables ([lines 49-52](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L49-L52)) for `CreditCheckRequestMessage`, `CreditCheckResponseMessage`, `ComplianceCheckRequestMessage`, and `ComplianceCheckResponseMessage`.
   - Neither [JavaAdapterProcess.wsdl](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.wsdl) nor [JavaAdapterProcess.xsd](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.xsd) defines these messages or corresponding XML schema elements.
5. **Incomplete `<componentType>` in `composite.xml`**:
   - In [composite.xml](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/composite.xml#L44-L48), the `LoanProcessOrchestration` component type only exposes `LoanProcessService`. It does not declare `<reference name="CreditCheckPartner">` or `<reference name="ComplianceCheckPartner">`, making wires at lines 62-70 structurally invalid in SCA.

### B. Severe Functional Bugs & Parameter Mismatches
6. **`DecisionEngine.java` Parameter Map Bug (DTI Never Calculated)**:
   - In [LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L185-L191), the BPEL code puts pre-calculated `"dti"` and `"ltv"` into the loan map:
     ```java
     loanData.put("loanAmount", loanAmount);
     loanData.put("propertyValue", propertyValue);
     loanData.put("dti", dti);
     loanData.put("ltv", ltv);
     ```
   - However, [DecisionEngine.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/DecisionEngine.java#L32-L40) **completely ignores** the `"dti"` and `"ltv"` keys in the map! Instead, it attempts to read `"monthlyIncome"` and `"monthlyDebt"` to calculate DTI:
     ```java
     double monthlyIncome = getDouble(loanData, "monthlyIncome", 0);
     double monthlyDebt = getDouble(loanData, "monthlyDebt", 0);
     double dti = (monthlyIncome > 0) ? (monthlyDebt / monthlyIncome) * 100 : 0;
     ```
   - Because `monthlyIncome` and `monthlyDebt` are never put into the map by BPEL, `monthlyIncome` is `0`, and **`dti` will ALWAYS evaluate to `0.0`**! As a consequence, high-debt applicants with extreme DTI will never be declined by the decision engine.
7. **Compliance Check Interface Parameter Mismatch**:
   - In [LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L210-L222), the assign prepares the compliance request with:
     ```xml
     <copy><from>$inputVariable.payload/types:loanAmount</from><to>$complianceCheckRequest.payload/types:loanAmount</to></copy>
     <copy><from>$decisionStatus</from><to>$complianceCheckRequest.payload/types:decisionStatus</to></copy>
     <copy><from>$dtiRatio</from><to>$complianceCheckRequest.payload/types:dtiRatio</to></copy>
     ```
   - But [IComplianceCheckRemote.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/IComplianceCheckRemote.java#L30) method signature is:
     `checkCompliance(double loanAmount, double dti, double ltv)`
   - The BPEL invocation passes `decisionStatus` instead of `ltv`, completely omitting `ltv` which is required for high-cost loan checks.
8. **EJB Data Field Name Mismatches**:
   - For Credit Check: BPEL expects `$creditCheckResponse.payload/types:creditScore`, but [CreditReport.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/CreditReport.java#L15) defines the property as `score` (`getScore()`), not `creditScore`.
   - For Compliance Check: BPEL expects `$complianceCheckResponse.payload/types:complianceStatus`, but [ComplianceResult.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/ComplianceResult.java#L20) defines the recommendation field as `recommendation`, not `complianceStatus`.
9. **XPath Schema Element Mismatch in `PrepareCreditCheckRequest`**:
   - [LoanProcessOrchestration.bpel](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L108-L115) copies `$inputVariable.payload/types:applicantFirstName` and `$inputVariable.payload/types:applicantLastName`.
   - In [JavaAdapterProcess.xsd](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.xsd#L15), the schema only defines `<xsd:element name="applicantName" type="xsd:string"/>`. There are no first/last name fields. This would trigger a BPEL `selectionFailure` runtime fault.

### C. Data Flow Disconnects & Discarded Results
10. **Credit Score Result is Discarded**:
    - In Step 1b ([line 127](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L127)), credit score is read into `$creditScore`.
    - In Step 3 ([line 190](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L190)), the decision engine hardcodes `loanData.put("creditScore", 720);`, completely ignoring the result of Step 1b.
    - In Step 4 ([line 256](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L256)), it fabricates another credit score: `creditScore = 600 + (riskScore * 2)`.
11. **Compliance Status Result is Never Used**:
    - In Step 3b ([line 233](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/LoanProcessOrchestration.bpel#L233)), `$complianceStatus` is stored, but it is **never assigned** to the response (`outputVariable`) or evaluated anywhere in the remainder of the process.
12. **Redundant XML Variable Parsing**:
    - Across `<bpelx:exec>` blocks in Steps 1, 3, and 4, BPEL executes `Double.parseDouble((String)getVariableData(...))` repeatedly for `loanAmount` and `propertyValue` instead of caching them in BPEL variables.
13. **Calculated Values Not Exposed in Response Schema**:
    - Step 4 calculates `interestRate` and Step 2 calculates `ocrConfidence`, but neither is mapped to `outputVariable` because [JavaAdapterProcess.xsd](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/JavaAdapterProcess.xsd#L26-L38) omits interest rate from `LoanProcessResponseType`.

### D. Java Client & JAX-WS Invocation Issues
14. **Non-Annotated SEI in `JavaAdapterClient.java`**:
    - In [JavaAdapterClient.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-client/src/com/soa/client/JavaAdapterClient.java#L32), the client invokes `service.getPort(JavaAdapterProcessPort.class)`.
    - However, `JavaAdapterProcessPort` ([lines 123-125](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-client/src/com/soa/client/JavaAdapterClient.java#L123-L125)) lacks standard `@WebService` / `@SOAPBinding` annotations, causing standard JAX-WS runtimes (Metro / CXF) to throw `WebServiceException: Class is not an Annotated SEI`.

### E. Divergence from Documentation
15. **`README.md` Contradiction**:
    - [README.md](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/README.md#L179-L180) states:
      > *"No `<reference>`: Java embedding is inline code, not external service"*
      > *"No partnerLink: Unlike `<invoke>`, `<bpelx:exec>` has no partnerLink"*
    - In reality, the composite includes 2 `<reference>` tags (`CreditCheckEJB`, `ComplianceCheckEJB`), 2 Spring components, and BPEL partner links with `<invoke>` calls. The project appears to be a hybrid created by partially merging an EJB/Spring test suite into the Java embedding test suite.

---

## 3. Unused Files & Dead Components

The following files and components exist in the workspace but are **completely disconnected** from the execution flow:

| Category | File / Component | Status / Reason |
| :--- | :--- | :--- |
| **Orphaned Spring Components** | [composite.xml](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/composite.xml#L77-L89): `RiskScorerComponent`, `DocClassifierComponent` | Declared in `composite.xml`, but have **zero wires** connecting them to any service or component. |
| **Orphaned Spring Contexts** | [doc-classifier-context.xml](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/spring/doc-classifier-context.xml)<br>[risk-scorer-context.xml](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapter-composite/spring/risk-scorer-context.xml) | Spring bean XMLs configured for SCA services, but unreachable by BPEL or client. |
| **Unused Spring Classes** | [IDocClassifier.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/spring/IDocClassifier.java)<br>[DocClassifierImpl.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/spring/DocClassifierImpl.java)<br>[IRiskScorer.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/spring/IRiskScorer.java)<br>[RiskScorerImpl.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/spring/RiskScorerImpl.java) | **Dead code**. Duplicates the logic in `OCRProcessor.java` and `DecisionEngine.java`. Never referenced anywhere in BPEL or Java client. |
| **Uncalled Adapter Methods** | [DTICalculator.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/DTICalculator.java#L54-L82): `calculateFrontEndDTI`, `calculateBackEndDTI`, `isQMCompliant` | Auxiliary calculation methods never invoked in BPEL. |
| **Uncalled Pricing Methods** | [AmortizationCalculator.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/adapters/AmortizationCalculator.java#L86-L140): `calculatePITI`, `calculateEntry`, inner class `AmortizationEntry` | Auxiliary calculation logic never invoked in BPEL. |
| **Uncalled EJB Methods** | [CreditCheckEJB.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/CreditCheckEJB.java#L49-L56): `getCreditScore`, `meetsMinimumScore`<br>[ComplianceCheckEJB.java](file:///d:/ramu/Personal/Hexaware/Rapidx_Freddie_Rerpos/LQA_Repositories/SOA_Repositories/soa-java-adapter-test-main/soa-java-adapter-test-main/java-adapters/src/com/soa/ejb/ComplianceCheckEJB.java#L82-L98): `isQualifiedMortgage`, `meetsTridTiming` | Interface methods defined and implemented, but never invoked by the BPEL orchestration. |
| **Missing Build Artifacts** | Missing `pom.xml` / `build.gradle` / `.jpr` | No build tool descriptors exist to compile the Java adapters into the JARs required under `SCA-INF/lib/`. |
