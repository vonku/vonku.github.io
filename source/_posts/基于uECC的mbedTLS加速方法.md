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

针对这一情况，常常使用uecc代替mbedtls的部分实现，来达到加速的效果。但如果应用希望保留使用mbedtls的接口，又要如何通过uecc加速呢？答案是对mbedtls的参数和uecc的参数进行转换，在内部替换掉mbedtls的部分实现。

### 1 uECC加速

#### 1.1 ecdh-生成公开参数

ecdh生成公开参数的函数是 `mbedtls_ecdh_gen_public()`，实际测试过程中发现该函数比较耗时。

它的输出的结果保存在 `d` 和 `Q`这两个参数中。因此我们使用 `uECC_make_key()` 产生公私钥对之后，再转换成对应的格式即可。

```c
#define ECDH_PUB_KEY_LEN        (64)
#define ECDH_PUB_KEY_XY_LEN     (32)
#define ECDH_PRI_KEY_LEN        (32)
#define ECDH_SHARED_KEY_LEN     (32)

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

#### 1.2 ecdh-生成共享密钥

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

#### 1.3 ecdsa-校验签名

ecdsa校验签名的过程中，发现是 `ecdsa_verify_restartable()` 这个方法比较耗时。我们需要使用`uECC_verify()`来代替 ``ecdsa_verify_restartable()`验签函数。

1. 根据 `&ctx->Q`和曲线参数转换成`pub_key_buf`
2. 将mbedtls签名转换成uECC的签名格式
3. 使用`uECC_verify()`验证签名

```c
#define SIG_MIN_LEN             (70)
#define SIG_MAX_LEN             (72)
#define SIG_LEN                 (64)

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

### 2 uECC加速实战

#### 2.1 ecdh加速实验

##### 2.1.1 ECDH共享参数

双方在进行ECDH密钥协商之前，需要选择相同的椭圆曲线方程、大素数p和生成元G。

（1）选择相同的大素数p；

（2）选择椭圆曲线方程E：y^2 = x^3+ax+b mod p

（3）选择相同的生成元G(Gx, Gy)。

在实际应用中，如果约定使用标准曲线如 secp256r1，那么双方就可以很容易地确认大素数P、椭圆曲线方程和生成元。

##### 2.1.2 密钥协商过程

（1）Alice选择一个比椭圆曲线的阶小的随机数da作为私密参数，该参数可以理解为Alice的私钥，计算Qa = daG = (xa, ya)，并把Qa发给Bob

（2）Bob选择一个比椭圆曲线的阶小的随机数db作为私密参数，该参数可以理解为Bob的私钥，计算Qb = dbG = (xb, yb)，并把Qb发给Alice

（3）Bob收到Qa后计算得到共享密钥Kb = dbQa = db(daG)

（4）Alice收到Qb后计算得到共享密钥Ka = daQb = da(dbG)

根据椭圆结合律，db(daG) = da(dbG) = （xq, yq）。即得到相同的共享密钥 Kb = Ka。实际使用可以是xq和yq两部分，也可以是xq单一部分。

##### 2.1.3 实验对比

测试代码：

```c
int main( int argc, char *argv[] )
{
    int ret = 1;
    int exit_code = MBEDTLS_EXIT_FAILURE;
    size_t olen;
    char buf[65];
    mbedtls_ecp_group grp;
    mbedtls_mpi cli_secret, srv_secret;
    mbedtls_mpi cli_pri, srv_pri;
    mbedtls_ecp_point cli_pub, srv_pub;

    mbedtls_entropy_context entropy;
    mbedtls_ctr_drbg_context ctr_drbg;
    const char pers[] = "ecdh";
    ((void) argc);
    ((void) argv);

    clock_t start_time, end_time;

    start_time = clock();

    for (int i = 0; i < 100; ++i) {

        mbedtls_mpi_init(&cli_pri );
        mbedtls_mpi_init(&srv_pri);
        mbedtls_mpi_init(&cli_secret);
        mbedtls_mpi_init(&srv_secret);

        mbedtls_ecp_group_init(&grp);
        mbedtls_ecp_point_init(&cli_pub);
        mbedtls_ecp_point_init(&srv_pub);

        mbedtls_entropy_init(&entropy);
        mbedtls_ctr_drbg_init( &ctr_drbg );

        /*
         * Initialize random number generation
         */
        mbedtls_printf( "  . Seeding the random number generator..." );
        fflush( stdout );
    
        if( ( ret = mbedtls_ctr_drbg_seed( &ctr_drbg, mbedtls_entropy_func, &entropy,
                                   (const unsigned char *) pers,
                                   sizeof pers ) ) != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ctr_drbg_seed returned %d\n", ret );
            goto exit;
        }

        mbedtls_printf( " ok\n" );

        /*
         * Client: inialize context and generate keypair
         */
        mbedtls_printf( "  . Setting up client context..." );
        fflush( stdout );

        ret = mbedtls_ecp_group_load( &grp, MBEDTLS_ECP_DP_SECP256R1);
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecp_group_load returned %d\n", ret );
            goto exit;
        }

        ret = mbedtls_ecdh_gen_public(&grp, &cli_pri, &cli_pub,
                                       mbedtls_ctr_drbg_random, &ctr_drbg );
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecdh_gen_public returned %d\n", ret );
            goto exit;
        }

        ret = mbedtls_ecp_point_write_binary(&grp, &cli_pub, MBEDTLS_ECP_PF_UNCOMPRESSED, &olen, buf, sizeof(buf) );
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecp_point_write_binary returned %d\n", ret );
            goto exit;
        }

        mbedtls_printf( " ok\n" );

        /*
         * Server: initialize context and generate keypair
         */
        mbedtls_printf( "  . Setting up server context..." );
        fflush( stdout );


        ret = mbedtls_ecdh_gen_public(&grp, &srv_pri, &srv_pub,
                                mbedtls_ctr_drbg_random, &ctr_drbg);
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecdh_gen_public returned %d\n", ret );
            goto exit;
        }

        ret = mbedtls_ecp_point_write_binary(&grp, &srv_pub, MBEDTLS_ECP_PF_UNCOMPRESSED, &olen, buf, sizeof(buf));
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_mpi_write_binary returned %d\n", ret );
            goto exit;
        }

        mbedtls_printf( " ok\n" );

        /*
         * Server: read peer's key and generate shared secret
         */
        mbedtls_printf( "  . Server reading client key and computing secret..." );
        fflush( stdout );

  
        ret = mbedtls_ecdh_compute_shared( &grp, &cli_secret,
                                           &srv_pub, &cli_pri,
                                           mbedtls_ctr_drbg_random, &ctr_drbg );
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecdh_compute_shared returned %d\n", ret );
            goto exit;
        }

        mbedtls_printf( " ok\n" );

        mbedtls_mpi_write_binary(&cli_secret, buf, mbedtls_mpi_size(&cli_secret));

        /*
         * Client: read peer's key and generate shared secret
         */
        mbedtls_printf( "  . Client reading server key and computing secret..." );
        fflush( stdout );

        ret = mbedtls_ecdh_compute_shared(&grp, &srv_secret,
            &cli_pub, &srv_pri,
            mbedtls_ctr_drbg_random, &ctr_drbg);
        if (ret != 0)
        {
            mbedtls_printf(" failed\n  ! mbedtls_ecdh_compute_shared returned %d\n", ret);
            goto exit;
        }

        mbedtls_printf(" ok\n");

        mbedtls_mpi_write_binary(&srv_secret, buf, mbedtls_mpi_size(&srv_secret));

        /*
         * Verification: are the computed secrets equal?
         */
        mbedtls_printf( "  . Checking if both computed secrets are equal..." );
        fflush( stdout );

        ret = mbedtls_mpi_cmp_mpi( &cli_secret, &srv_secret);
        if( ret != 0 )
        {
            mbedtls_printf( " failed\n  ! mbedtls_ecdh_compute_shared returned %d\n", ret );
            goto exit;
        }

        mbedtls_printf( " ok\n" );

        exit_code = MBEDTLS_EXIT_SUCCESS;

    exit:

        mbedtls_mpi_free(&cli_pri);
        mbedtls_mpi_free(&srv_pri);
        mbedtls_mpi_free(&cli_secret);
        mbedtls_mpi_free(&srv_secret);
        mbedtls_ecp_group_free(&grp);
        mbedtls_ecp_point_free(&cli_pub);
        mbedtls_ecp_point_free(&srv_pub);

        mbedtls_ctr_drbg_free( &ctr_drbg );
        mbedtls_entropy_free( &entropy );

    }

    end_time = clock();

    double time_cost = (double)(end_time - start_time) / CLOCKS_PER_SEC;

    mbedtls_printf("ecdh %f seconds with uecc.\n", time_cost);

#if defined(_WIN32)
    mbedtls_printf("  + Press Enter to exit this program.\n");
    fflush(stdout); getchar();
#endif

    return( exit_code );
}
```

ecdh的实验结果如下：

```
ecdh 18.827000 seconds without uecc.
ecdh 13.479000 seconds with uecc
```

ecdsa的结果如下：

```
ecdsa 21.767000 seconds without uecc.
ecdsa 11.567000 seconds with uecc
```

这是在windows平台上测试的结果。实际在嵌入式平台上的结果是接近10倍的优化。

**参考资料**：

1. *https://github.com/kmackay/micro-ecc （static版本）*

