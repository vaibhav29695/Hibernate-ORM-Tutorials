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
