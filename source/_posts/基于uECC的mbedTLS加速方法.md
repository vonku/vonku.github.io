---
title: 基于uECC的mbedTLS加速方法
date: 2023-04-29 19:43:00
categories: Security
tags: 
- 物联网安全
- mbedtls
- uECC

---

> 主要内容：本文主要介绍使用uECC加速mbedTLS ecdh和ecdsa流程的方法。

<!--more-->

mbedtls是被广泛使用的开源安全加密库，在实际使用过程中发现，ecdh和ecdsa的软件计算过程耗时比较长。

针对这一情况，常常使用uecc代替mbedtls的部分实现，来达到加速的效果。但如果应用希望保留使用mbedtls的接口，又要如何通过uecc加速呢？答案是对mbedtls的参数和uecc的参数进行转换，在内部hook掉mbedtls的部分实现。

### 1 ecdh-产生公私钥对

产生公私钥对的耗时方法是 `mbedtls_ecdh_gen_public()`，输出的结果保存在 `d` 和 `Q`这两个参数中。因此我们使用 `uECC_make_key()` 产生公私钥对之后，再转换成对应的格式即可。

```c
int mbedtls_ecdh_gen_public( mbedtls_ecp_group *grp, mbedtls_mpi *d, mbedtls_ecp_point *Q,
                     int (*f_rng)(void *, unsigned char *, size_t),
                     void *p_rng )
{
  ... 

#if UECC_ACC
    unsigned char priKey[ECDH_PRI_KEY_LEN];
    unsigned char pub_key_buf[ECDH_PUB_KEY_LEN + 1];
    unsigned char *pubKey = pub_key_buf + 1;

    if (grp->id == MBEDTLS_ECP_DP_SECP256R1) {
        if (uECC_make_key(pubKey, priKey)) {
            pub_key_buf[0] = 0x04;
            mbedtls_ecp_point_read_binary(grp, Q, pub_key_buf, ECDH_PUB_KEY_LEN + 1);
            mbedtls_mpi_read_binary(d, priKey, ECDH_PRI_KEY_LEN);
            return 0;
        }
    }
#endif

    return( ecdh_gen_public_restartable( grp, d, Q, f_rng, p_rng, NULL ) );
}
```

### 2 ecdh-生成共享密钥

`mbedtls_ecdh_compute_shared()`生成共享密钥的时候，输入参数是`Q`和`d`，生成的共享密钥输出到`z`。因此我们先从`Q`得到uECC所需要的`pubKey`，从`d`得到uECC所需要的`priKey`；然后使用`uECC_shared_secret()`得到sharedKey之后再转换成mbedtls所需要的`z`即可。

```c
int mbedtls_ecdh_compute_shared( mbedtls_ecp_group *grp, mbedtls_mpi *z,
                         const mbedtls_ecp_point *Q, const mbedtls_mpi *d,
                         int (*f_rng)(void *, unsigned char *, size_t),
                         void *p_rng )
{
    ... 

#if UECC_ACC
    if (grp->id == MBEDTLS_ECP_DP_SECP256R1) {
        unsigned char priKey[ECDH_PRI_KEY_LEN];
        unsigned char pub_key_buf[ECDH_PUB_KEY_LEN + 1];
        unsigned char sharedKey[ECDH_SHARED_KEY_LEN];
        unsigned char *pubKey = pub_key_buf + 1;
        size_t len;

        mbedtls_ecp_point_write_binary(grp, Q, MBEDTLS_ECP_PF_UNCOMPRESSED, &len, pub_key_buf, sizeof(pub_key_buf));
        mbedtls_mpi_write_binary(d, priKey, ECDH_PRI_KEY_LEN);
        if (uECC_shared_secret(pubKey, priKey, sharedKey)) {
            mbedtls_mpi_read_binary(z, sharedKey, ECDH_SHARED_KEY_LEN);
            return 0;
        }
    }
#endif
    return( ecdh_compute_shared_restartable( grp, z, Q, d,
                                             f_rng, p_rng, NULL ) );
}
```

### 3 ecdsa-校验签名

ecdsa校验签名的过程中，发现是 `ecdsa_verify_restartable()` 这个方法比较耗时。我们需要使用`uECC_verify()`来代替 ``ecdsa_verify_restartable()`验签函数。

1. 根据 `&ctx->Q`和曲线参数转换成`pub_key_buf`
2. 将mbedtls签名转换成uECC的签名格式
3. 使用`uECC_verify()`验证签名

```c
int mbedtls_ecdsa_read_signature_restartable( mbedtls_ecdsa_context *ctx,
                          const unsigned char *hash, size_t hlen,
                          const unsigned char *sig, size_t slen,
                          mbedtls_ecdsa_restart_ctx *rs_ctx )
{
  ...
#if UECC_ACC
    uint8_t pub_key_buf[ECDH_PUB_KEY_LEN + 1];
    uint8_t sig_buf[SIG_LEN];
    uint8_t *pubKey = pub_key_buf + 1;
#endif
  ...
#if UECC_ACC
    if  (ctx->grp.id == MBEDTLS_ECP_DP_SECP256R1) {
        mbedtls_ecp_point_write_binary(&ctx->grp, &ctx->Q, MBEDTLS_ECP_PF_UNCOMPRESSED, &len, \
                                        pub_key_buf, sizeof(pub_key_buf));

        if ( ( ret = mbedtls_ecdsa_get_uecc_sig(sig, slen, sig_buf) ) != 0 )
            goto cleanup;

        ret = uECC_verify(pubKey, hash, sig_buf) ? 0 : MBEDTLS_ERR_ECP_VERIFY_FAILED;
        if (ret)
            goto cleanup;
    } else {
        if( ( ret = ecdsa_verify_restartable( &ctx->grp, hash, hlen,
                                  &ctx->Q, &r, &s, rs_ctx ) ) != 0 )
            goto cleanup;
    }
#else
    if( ( ret = ecdsa_verify_restartable( &ctx->grp, hash, hlen,
                              &ctx->Q, &r, &s, rs_ctx ) ) != 0 )
        goto cleanup;
#endif
}
```

其中，`mbedtls_ecdsa_get_uecc_sig()`是将mbedtls的asn1格式的签名数据，转换成uECC的 `r+s` 的签名格式：

```c
/**
 * asn1 sig : 30 + LEN1 + 02 + LEN2 + 00(optional) + r + 02 + LEN3 + 00(optional) + s
*/
int mbedtls_ecdsa_get_uecc_sig(const unsigned char *sig, size_t slen, unsigned char *uecc_sig)
{
    ECDSA_VALIDATE_RET( sig  != NULL );
    ECDSA_VALIDATE_RET( uecc_sig != NULL );
    size_t offset = 0;
    const size_t asn1_hdr_len = 3;
    size_t padding = 0;

    memset (uecc_sig, 0, 2*uECC_BYTES);

    offset += asn1_hdr_len;

    /* get r */
    if (sig[offset] <= uECC_BYTES) {
        padding = uECC_BYTES - sig[offset];
        offset += 1;    /* len2 */
        memcpy(uecc_sig + padding, sig + offset, uECC_BYTES - padding);
        offset += (uECC_BYTES - padding + 1);
    } else if (sig[offset] == uECC_BYTES + 1) {
        offset += 2;   /* len2|00 */
        memcpy(uecc_sig, sig + offset, uECC_BYTES);
        offset += (uECC_BYTES + 1);
    } else {
        return MBEDTLS_ERR_ECP_BAD_INPUT_DATA;
    }

    /* get s */
    if (sig[offset] <= uECC_BYTES) {
        padding = uECC_BYTES - sig[offset];
        offset += 1;    /* len2 */
        memcpy(uecc_sig + uECC_BYTES + padding, sig + offset, uECC_BYTES - padding);
    }  else if (sig[offset] == uECC_BYTES + 1) {
        offset += 2;    /* len3|00 */
        memcpy(uecc_sig + uECC_BYTES, sig + offset, uECC_BYTES);
    } else {
        return MBEDTLS_ERR_ECP_BAD_INPUT_DATA;
    }

    return 0;
}
```

