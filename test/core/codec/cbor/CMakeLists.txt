
addtest(cbor_test
    cbor_test.cpp
    )
target_link_libraries(cbor_test
    cbor
    hexutil
    ${GV_nlp_p2p_libs}
    )

addtest(cbor_buffering_test
    cbor_buffering_test.cpp
    )
target_link_libraries(cbor_buffering_test
    cbor_stream
    )
