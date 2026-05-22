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
