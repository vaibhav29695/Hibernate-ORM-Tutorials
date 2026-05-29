private BigDecimal calculateGstAmount(
        MerchantPricingResponse pricingStructure,
        MerchantPricingDto dto) {

    if (pricingStructure.getServiceTax() == null) {
        return BigDecimal.ZERO;
    }

    if (dto.getAggServiceFeeAb() == null) {
        return BigDecimal.ZERO;
    }

    if ("P".equalsIgnoreCase(
            pricingStructure.getServiceTaxType())) {

        return dto.getAggServiceFeeAb()
                .multiply(pricingStructure.getServiceTax())
                .divide(BigDecimal.valueOf(100),
                        2,
                        RoundingMode.HALF_UP);
    }

    return pricingStructure.getServiceTax();
}




MerchantPricingDto merchantPricingDto =
        calculateFee(pricingStructure,
                     merchantPricingRequest);

BigDecimal gstAmount =
        calculateGST(
                pricingStructure,
                merchantPricingDto);

merchantPricingDto.setGstAmount(gstAmount);

checkAtrnAndSavePricingInfo(
        merchantPricingRequest,
        merchantPricingDto,
        pricingStructure);

return buildMerchantFeeResponse(
        merchantPricingRequest,
        merchantPricingDto);





private BigDecimal calculateGST(
        MerchantPricingResponse pricingStructure,
        MerchantPricingDto pricingDto) {

    if (pricingStructure.getServiceTax() == null) {
        return BigDecimal.ZERO;
    }

    if (pricingDto.getAggServiceFeeAb() == null) {
        return BigDecimal.ZERO;
    }

    return pricingDto.getAggServiceFeeAb()
            .multiply(pricingStructure.getServiceTax())
            .divide(BigDecimal.valueOf(100),
                    2,
                    RoundingMode.HALF_UP);
}









import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.text.MessageFormat;
import java.util.Optional;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderDaoTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderDao orderDao;

    private TransactionDto transactionDto;
    private Order order;

    @BeforeEach
    void setUp() {

        transactionDto = new TransactionDto();
        transactionDto.setSbiOrderRefNumber("SBI123");
        transactionDto.setOrderRefNumber("ORD123");

        order = new Order();
    }

    @Test
    void testGetOrderDetails_Success() {

        when(orderRepository
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber()))
                .thenReturn(Optional.of(order));

        Order result = orderDao.getOrderDetails(transactionDto);

        assertNotNull(result);
        assertEquals(order, result);

        verify(orderRepository, times(1))
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber());
    }

    @Test
    void testGetOrderDetails_OrderNotFound() {

        when(orderRepository
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber()))
                .thenReturn(Optional.empty());

        PaymentException exception = assertThrows(
                PaymentException.class,
                () -> orderDao.getOrderDetails(transactionDto));

        assertEquals(
                ErrorConstants.NOT_FOUND_ERROR_CODE,
                exception.getErrorCode());

        assertEquals(
                MessageFormat.format(
                        ErrorConstants.NOT_FOUND_ERROR_MESSAGE,
                        "Order"),
                exception.getMessage());

        verify(orderRepository, times(1))
                .findBySbiOrderRefNumberAndOrderRefNumber(
                        transactionDto.getSbiOrderRefNumber(),
                        transactionDto.getOrderRefNumber());
    }
}








import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.nio.charset.StandardCharsets;
import java.sql.Timestamp;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OfflinePaymentDaoTest {

    @Mock
    private OfflinePaymentRepository offlinePaymentRepository;

    @Mock
    private CashManagementMapper mapper;

    @InjectMocks
    private OfflinePaymentDao offlinePaymentDao;

    @Mock
    private TransactionDto transactionDto;

    @Mock
    private PlainCashChallanJsonRequest plainCashChallanJsonRequest;

    @Mock
    private CashChallanPaymentFinalResponseDto cashChallanPaymentFinalResponseDto;

    @Mock
    private NeftPaymentFinalResponseDto neftPaymentFinalResponseDto;

    @Mock
    private CashManagementDto cashManagementDto;

    private UUID reportManagementId;

    @BeforeEach
    void setUp() {

        reportManagementId = UUID.randomUUID();

        when(transactionDto.getMerchantId()).thenReturn(1001L);
        when(transactionDto.getAtrnNum()).thenReturn("ATRN12345");
        when(transactionDto.getPayMode()).thenReturn("CASH");

        when(plainCashChallanJsonRequest.getNameOfTheCustomer())
                .thenReturn("Vaibhav");

        when(plainCashChallanJsonRequest.getMobileNumber())
                .thenReturn("9876543210");

        when(plainCashChallanJsonRequest.getEmailId())
                .thenReturn("test@gmail.com");

        when(plainCashChallanJsonRequest.getChallanGenerationDateAndTime())
                .thenReturn(new Timestamp(System.currentTimeMillis()));

        when(plainCashChallanJsonRequest.getChallanExpiryOn())
                .thenReturn(new Timestamp(System.currentTimeMillis()));

        when(cashManagementDto.getAtrnNum())
                .thenReturn("ATRN12345");
    }

    @Test
    void testSaveCashPdfDetails() {

        assertDoesNotThrow(() ->
                offlinePaymentDao.saveCashPdfDetails(
                        reportManagementId,
                        ReportStatus.SUCCESS,
                        transactionDto,
                        cashChallanPaymentFinalResponseDto,
                        plainCashChallanJsonRequest
                ));

        verify(offlinePaymentRepository, times(1))
                .save(any(OfflinePaymentModeDetailsEntity.class));

        verify(mapper, times(1))
                .mapEntityToDto(any(OfflinePaymentModeDetailsEntity.class));
    }

    @Test
    void testFindByAtrnNum() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        CashManagementDto dto = new CashManagementDto();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        when(mapper.mapEntityToDto(entity))
                .thenReturn(dto);

        offlinePaymentDao.findByAtrnNum("ATRN12345");

        verify(offlinePaymentRepository, times(1))
                .findByAtrnNum("ATRN12345");

        verify(mapper, times(1))
                .mapEntityToDto(entity);
    }

    @Test
    void testUpdateCashPdfDetails() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfDetails(
                        "test.pdf",
                        cashManagementDto
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testUpdateCashPdfBlobDetails() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfBlobDetails(
                        "BASE64DATA",
                        cashManagementDto,
                        "test.pdf"
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testUpdateCashPdfBlobData() {

        OfflinePaymentModeDetailsEntity entity =
                new OfflinePaymentModeDetailsEntity();

        byte[] fileData = "PDF_DATA".getBytes(StandardCharsets.UTF_8);

        when(offlinePaymentRepository.findByAtrnNum("ATRN12345"))
                .thenReturn(entity);

        assertDoesNotThrow(() ->
                offlinePaymentDao.updateCashPdfBlobData(
                        fileData,
                        cashManagementDto,
                        "test.pdf"
                ));

        verify(offlinePaymentRepository, times(1))
                .save(entity);
    }

    @Test
    void testSaveNeftPdfDetails() {

        assertDoesNotThrow(() ->
                offlinePaymentDao.saveNeftPdfDetails(
                        reportManagementId,
                        ReportStatus.SUCCESS,
                        transactionDto,
                        neftPaymentFinalResponseDto,
                        plainCashChallanJsonRequest
                ));

        verify(offlinePaymentRepository, times(1))
                .save(any(OfflinePaymentModeDetailsEntity.class));

        verify(mapper, times(1))
                .mapEntityToDto(any(OfflinePaymentModeDetailsEntity.class));
    }
}









import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertDoesNotThrow;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.doThrow;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.lang.reflect.Method;
import java.util.UUID;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class NotificationDaoTest {

    @InjectMocks
    private NotificationDao notificationDao;

    @Mock
    private NotificationMapper notificationMapper;

    @Mock
    private NotificationManagementRepository notificationManagementRepository;

    @Mock
    private SmsNotificationProducer smsNotificationProducer;

    @Mock
    private EmailNotificationProducer emailNotificationProducer;

    @Mock
    private EmailService emailService;

    @Mock
    private CashChallanConfigDetails cashChallanConfigDetails;

    @Mock
    private SmsService smsService;

    @BeforeEach
    void setUp() {
    }

    @Test
    void testPublishSmsNotification() {

        CashSmsDto cashSmsDto = new CashSmsDto();
        cashSmsDto.setRequestType("SMS");

        String routingKey = "sms-routing";

        notificationDao.publishSmsNotification(cashSmsDto, routingKey);

        verify(smsNotificationProducer)
                .publish(cashSmsDto.getRequestType(), routingKey, cashSmsDto);
    }

    @Test
    void testSendSmsNotification() {

        CashSmsDto cashSmsDto = new CashSmsDto();

        SmsDto smsDto = new SmsDto();

        when(notificationMapper.mapSmsDtoToToDto(any(CashSmsDto.class)))
                .thenReturn(smsDto);

        assertDoesNotThrow(() ->
                notificationDao.sendSmsNotification(cashSmsDto));

        verify(notificationMapper)
                .mapSmsDtoToToDto(cashSmsDto);

        verify(smsService)
                .sendSMS(smsDto);
    }

    @Test
    void testSendSMS_Exception() throws Exception {

        SmsDto smsDto = new SmsDto();

        doThrow(new RuntimeException("SMS Exception"))
                .when(smsService)
                .sendSMS(any(SmsDto.class));

        Method method = NotificationDao.class
                .getDeclaredMethod("sendSMS", SmsDto.class);

        method.setAccessible(true);

        method.invoke(notificationDao, smsDto);

        // verify service called
        verify(smsService).sendSMS(smsDto);
    }

    @Test
    void testBuildNotificationManagement_ForSms() throws Exception {

        CashSmsDto cashSmsDto = new CashSmsDto();

        cashSmsDto.setRequestType("SMS");

        UUID entityId = UUID.randomUUID();
        cashSmsDto.setEntityId(entityId);

        Method method = NotificationDao.class
                .getDeclaredMethod("buildNotificationManagement",
                        CashSmsDto.class);

        method.setAccessible(true);

        NotificationManagement result =
                (NotificationManagement) method.invoke(notificationDao,
                        cashSmsDto);

        assertEquals("SMS", result.getRequestType());
        assertEquals(entityId, result.getEntityId());
    }

    @Test
    void testPublishEmailNotification() {

        CashEmailDto cashEmailDto = new CashEmailDto();

        cashEmailDto.setRequestType("EMAIL");

        UUID entityId = UUID.randomUUID();

        String routingKey = "email-routing";

        notificationDao.publishEmailNotification(
                cashEmailDto,
                routingKey,
                entityId
        );

        verify(emailNotificationProducer)
                .publish(
                        cashEmailDto.getRequestType(),
                        routingKey,
                        cashEmailDto
                );
    }

    @Test
    void testGetEmailDto_WhenRecipientPresent() throws Exception {

        CashEmailDto cashEmailDto = new CashEmailDto();

        UUID userId = UUID.randomUUID();

        when(cashChallanConfigDetails.getFrom())
                .thenReturn("test@gmail.com");

        when(cashChallanConfigDetails.getRecipient())
                .thenReturn("receiver@gmail.com");

        Method method = NotificationDao.class
                .getDeclaredMethod(
                        "getEmailDto",
                        CashEmailDto.class,
                        UUID.class
                );

        method.setAccessible(true);

        method.invoke(notificationDao, cashEmailDto, userId);

        assertEquals("test@gmail.com", cashEmailDto.getFrom());
        assertEquals(userId, cashEmailDto.getEntityId());
        assertEquals("receiver@gmail.com", cashEmailDto.getRecipient());
    }

    @Test
    void testSendEmailNotification() {

        CashEmailDto cashEmailDto = new CashEmailDto();

        EmailDto emailDto = new EmailDto();

        when(notificationMapper.mapEmailDtoToDto(any(CashEmailDto.class)))
                .thenReturn(emailDto);

        assertDoesNotThrow(() ->
                notificationDao.sendEmailNotification(cashEmailDto));

        verify(notificationMapper)
                .mapEmailDtoToDto(cashEmailDto);

        verify(emailService)
                .sendEmail(emailDto);
    }

    @Test
    void testSendEmail_Exception() throws Exception {

        EmailDto emailDto = new EmailDto();

        doThrow(new RuntimeException("Email Exception"))
                .when(emailService)
                .sendEmail(any(EmailDto.class));

        Method method = NotificationDao.class
                .getDeclaredMethod("sendEmail", EmailDto.class);

        method.setAccessible(true);

        method.invoke(notificationDao, emailDto);

        verify(emailService).sendEmail(emailDto);
    }
}









@Test
void testProcessInbDoubleVerRequest_StringResponse() {

    NbMapWebResponseDto nbMapWebResponseDto = new NbMapWebResponseDto();
    NbMapDVResponseDto nbMapDVResponseDto = new NbMapDVResponseDto();

    NbMapWebResponseDto response =
            nbPaymentDao.processInbDoubleVerRequest(
                    nbMapWebResponseDto,
                    nbMapDVResponseDto,
                    "encryptedResponse");

    assertNotNull(response);
}







import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

import java.math.BigDecimal;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class NbPaymentDaoTest {

    @Mock
    private PaymentDao paymentDao;

    @Mock
    private InbEncryptionDecryptionUtil inbEncryptionDecryptionUtil;

    @Mock
    private PaymentValidator paymentValidator;

    @Mock
    private StatusUpdatePaymentDao statusUpdatePaymentDao;

    @InjectMocks
    private NbPaymentDao nbPaymentDao;

    private NbMapWebResponseDto nbMapWebResponseDto;
    private NbMapWebResponseDto nbMapDVResponseDto;

    @BeforeEach
    void setUp() {

        nbMapWebResponseDto = new NbMapWebResponseDto();
        nbMapWebResponseDto.setTxnrefNo("TXN123");
        nbMapWebResponseDto.setSbirefNo("SBI123");
        nbMapWebResponseDto.setAmount(new BigDecimal("100"));

        nbMapDVResponseDto = new NbMapWebResponseDto();
        nbMapDVResponseDto.setTxnrefNo("TXN123");
        nbMapDVResponseDto.setSbirefNo("SBI123");
        nbMapDVResponseDto.setAmount(new BigDecimal("100"));
        nbMapDVResponseDto.setCheckSum("checksum");
    }

    @Test
    void testProcessInbDoubleVerRequest_Success() {

        nbMapDVResponseDto.setStatus(PaymentConstants.SUCCESS_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentSuccessPendingStatusUpdate(
                        eq(PaymentStatus.SUCCESS),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_Pending() {

        nbMapDVResponseDto.setStatus(PaymentConstants.PENDING_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentSuccessPendingStatusUpdate(
                        eq(PaymentStatus.PENDING),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_Failure() {

        nbMapDVResponseDto.setStatus(PaymentConstants.FAILURE_CONST);

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testProcessInbDoubleVerRequest_DefaultCase() {

        nbMapDVResponseDto.setStatus("UNKNOWN");

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(true);

        NbMapWebResponseDto response =
                nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse");

        assertNotNull(response);

        verify(statusUpdatePaymentDao, times(1))
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        any());
    }

    @Test
    void testValidateCallBackResponse_ChecksumFail() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testValidateCallBackResponse_AmountMismatch() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testValidateCallBackResponse_BankRefMismatch() {

        when(paymentValidator.checkSumValidation(anyString(), anyString()))
                .thenReturn(true);

        when(paymentValidator.validateWebAndDvAmt(any(), any()))
                .thenReturn(true);

        when(paymentValidator.validateBankReferenceNumber(anyString(), anyString()))
                .thenReturn(false);

        assertThrows(
                PaymentException.class,
                () -> nbPaymentDao.processNbDoubleVerRequest(
                        nbMapWebResponseDto,
                        nbMapDVResponseDto,
                        "encryptedResponse"));
    }

    @Test
    void testProcessInbDoubleVerRequest_StringResponse() {

        String response =
                nbPaymentDao.processInbDoubleVerRequest(nbMapWebResponseDto);

        assertNotNull(response);
    }
}




import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import com.epay.payment.constant.PaymentConstants;
import com.epay.payment.dao.MobikwikWalletPaymentDao;
import com.epay.payment.dao.PaymentDao;
import com.epay.payment.dao.StatusUpdatePaymentDao;
import com.epay.payment.dto.MobikwikWalletDvResponseDto;
import com.epay.payment.dto.MobikwikWalletMapWebResponseDto;
import com.epay.payment.util.MobikwikWalletEncryptionDecryptionUtil;
import com.epay.payment.util.PaymentUtil;
import com.epay.payment.validator.PaymentValidator;
import com.epay.payment.config.WalletConfig;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class MobikwikWalletPaymentDaoTest {

    @Mock
    private PaymentDao paymentDao;

    @Mock
    private MobikwikWalletEncryptionDecryptionUtil walletEncryptionDecryptionUtil;

    @Mock
    private PaymentValidator paymentValidator;

    @Mock
    private StatusUpdatePaymentDao statusUpdatePaymentDao;

    @Mock
    private WalletConfig walletConfig;

    @Mock
    private PaymentUtil paymentUtil;

    @InjectMocks
    private MobikwikWalletPaymentDao mobikwikWalletPaymentDao;

    private MobikwikWalletMapWebResponseDto webResponseDto;
    private MobikwikWalletDvResponseDto dvResponseDto;

    @BeforeEach
    void setUp() {

        webResponseDto = new MobikwikWalletMapWebResponseDto();
        webResponseDto.setMid("MID123");
        webResponseDto.setOrderid("ORD123");

        dvResponseDto = new MobikwikWalletDvResponseDto();
        dvResponseDto.setOrderid("ORD123");
        dvResponseDto.setTxid("TXN123");
    }

    @Test
    void testProcessWalletDoubleVerRequest_StringResponse() {

        Mockito.spy(mobikwikWalletPaymentDao);

        String expected = "CALLBACK_RESPONSE";

        when(mobikwikWalletPaymentDao.getCallBackResponse("MID123", "ORD123"))
                .thenReturn(expected);

        String actual =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(webResponseDto);

        assertEquals(expected, actual);
    }

    @Test
    void testGetCallBackResponse() {

        when(walletConfig.getSecretKey()).thenReturn("SECRET_KEY");

        String response =
                mobikwikWalletPaymentDao.getCallBackResponse("MID123", "ORD123");

        verify(paymentDao).saveRequestLog(
                anyString(),
                anyString(),
                anyString(),
                anyString());

        assertEquals(response.contains("ORD123"), true);
    }

    @Test
    void testProcessWalletDoubleVerRequest_SuccessCase() {

        dvResponseDto.setStatuscode(
                PaymentConstants.WALLET_STATUS_SUCCESS_CONST);

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(paymentDao).saveResponseLog(
                anyString(),
                anyString(),
                anyString(),
                anyString(),
                anyString());

        verify(statusUpdatePaymentDao)
                .paymentSuccessPendingStatusUpdateGen(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals(
                PaymentConstants.WALLET_STATUS_SUCCESS_CONST,
                response.getStatuscode());
    }

    @Test
    void testProcessWalletDoubleVerRequest_FailureCase() {

        dvResponseDto.setStatuscode(
                PaymentConstants.FAILURE_CONST);

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(statusUpdatePaymentDao)
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals(
                PaymentConstants.FAILURE_CONST,
                response.getStatuscode());
    }

    @Test
    void testProcessWalletDoubleVerRequest_DefaultCase() {

        dvResponseDto.setStatuscode("UNKNOWN");

        MobikwikWalletDvResponseDto response =
                mobikwikWalletPaymentDao.processWalletDoubleVerRequest(
                        webResponseDto,
                        dvResponseDto,
                        "DECRYPT_RESPONSE");

        verify(statusUpdatePaymentDao)
                .paymentFailureStatusUpdate(
                        anyString(),
                        anyString(),
                        anyString(),
                        anyString());

        assertEquals("UNKNOWN", response.getStatuscode());
    }
}



import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.text.MessageFormat;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class MerchantPricingDaoTest {

    @Mock
    private MerchantOrderHybridFeeRepository merchantOrderHybridFeeRepository;

    @Mock
    private TransactionMapper transactionMapper;

    @InjectMocks
    private MerchantPricingDao merchantPricingDao;

    @Test
    void testGetTransactionAckReq_Success() {

        String atrn = "ATRN123";

        MerchantOrderHybridFee merchantOrderHybridFee = new MerchantOrderHybridFee();
        MerchantPricingDto merchantPricingDto = new MerchantPricingDto();

        when(merchantOrderHybridFeeRepository.findByAtrnNum(atrn))
                .thenReturn(Optional.of(merchantOrderHybridFee));

        when(transactionMapper.mapToMerchantPricingDto(merchantOrderHybridFee))
                .thenReturn(merchantPricingDto);

        MerchantPricingDto response = merchantPricingDao.getTransactionAckReq(atrn);

        assertEquals(merchantPricingDto, response);

        verify(merchantOrderHybridFeeRepository)
                .findByAtrnNum(atrn);

        verify(transactionMapper)
                .mapToMerchantPricingDto(merchantOrderHybridFee);
    }

    @Test
    void testGetTransactionAckReq_ATRNNotFound() {

        String atrn = "INVALID_ATRN";

        when(merchantOrderHybridFeeRepository.findByAtrnNum(atrn))
                .thenReturn(Optional.empty());

        PaymentException exception = assertThrows(
                PaymentException.class,
                () -> merchantPricingDao.getTransactionAckReq(atrn));

        assertEquals(
                ErrorConstants.INVALID_ERROR_CODE,
                exception.getErrorCode());

        assertEquals(
                MessageFormat.format(
                        ErrorConstants.INVALID_ERROR_MESSAGE,
                        PaymentConstants.atrn,
                        PaymentConstants.ATRN_NOT_FOUND),
                exception.getMessage());
    }
}
@Component
@RequiredArgsConstructor
public class MerchantPricingDao {

    private final MerchantOrderHybridFeeRepository merchantOrderHybridFeeRepository;
    private final TransactionMapper transactionMapper;

    public MerchantPricingDto getTransactionAckReq(String atrn) {

        MerchantOrderHybridFee merchantOrderHybridFee = merchantOrderHybridFeeRepository.findByAtrnNum(atrn)
                .orElseThrow(() -> new PaymentException(ErrorConstants.INVALID_ERROR_CODE, MessageFormat
                        .format(ErrorConstants.INVALID_ERROR_MESSAGE, PaymentConstants.atrn, PaymentConstants.ATRN_NOT_FOUND)));
        return transactionMapper.mapToMerchantPricingDto(merchantOrderHybridFee);
    }
}
