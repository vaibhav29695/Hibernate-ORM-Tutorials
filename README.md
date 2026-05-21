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
