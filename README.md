# frete-pagina-produto-wc
Inclui a simulação de frete na página do produto


### Para adicionar o calculo de frete junto ao produto basta alterar 3 arquivos. Vamos lá:

#### 1 - Altere o arquivo "_cdn/widgets/ecommerce/cat.add.php"
    - logo após o fechamento do <form> na linha 24, ACRESCENTE essas linhas:
```php
    <div class='wc_cart_total_shipment m_top'>
        <p>Frete:</p><input type='text' value='<?= (!empty($_SESSION['wc_shipment_zip']) ? $_SESSION['wc_shipment_zip'] : ''); ?>' class='formCep wc_cart_ship_val'/><button class='wc_cart_ship'>Calcular</button><img alt='Calculando Frete!' title='Calculando Frete!' src='<?= BASE; ?>/_cdn/widgets/ecommerce/load_g.gif'/>
        <div class='wc_cart_total_shipment_result'></div>
    </div>
```

#### 2 - Altere o arquivo "_cdn/widgets/ecommerce/cat.js"
##### - procure por "//SHIPMENT SELECT" e alter, depois altere a função "wcZipRecalculate", e deixe assim
```js
    //SHIPMENT SELECT
    $('html').on('click', '.wc_shipment', function () {
        $('.wc_cart_manager').fadeIn();
        var shipprice = $(this).val();
        var shipcode = $(this).attr('id');
        
        var item = ($('input[name="pdt_id"]').val() ? $('input[name="pdt_id"]').val() : ''); // LINHA NOVA
        var itemAmount = $('input[name="item_amount"]').val(); // LINHA NOVA

        $.post(action, {action: 'cart_shipment_select', wc_shipcode: shipcode, wc_shipprice: shipprice, pdtId: item, itemAmount: itemAmount}, function (data) { // LINHA NOVA
            $('.wc_cart_total span').html(data.cart_total);
            $('.wc_cart_shiping span').html(data.cart_ship);
            $('.wc_cart_price span').html(data.cart_price);
            $('.wc_cart_manager').fadeOut();
        }, 'json');
    });


    //SHIPMENT CALC
    function wcZipRecalculate(Shipvalue) {
        var Shipment = (Shipvalue ? Shipvalue : $('.wc_cart_ship_val').val());
        if (Shipment) {
            var item = ($('input[name="pdt_id"]').val() ? $('input[name="pdt_id"]').val() : '');
            var itemAmount = $('input[name="item_amount"]').val();
            var input = $('.wc_cart_ship_val');
            $.post(action, {action: 'cart_shipment', zipcode: Shipment, pdtId: item, itemAmount: itemAmount}, function (data) {
                $('.wc_cart_total_shipment_result').html(data.cart_shipment);
                $('.wc_cart_total_shipment img').fadeOut();
                $('.wc_cart_total_shipment_tag').fadeIn(0);

                if (data.trigger) {
                    wcCartTrigger(data.trigger);
                }

                if (data.reset) {
                    input.val('');
                }
                $('.wc_cart_manager').fadeOut();
            }, 'json');
        }
    }
```

##### - Procure por "//PLUS ITEM AMOUNT" linha 112, e "//LESS ITEM AMOUNT", e acrescente a chamada da função "wcZipRecalculate();" logo acima do "return false;" de cada função;
```js
    //PLUS ITEM AMOUNT
    $('.wc_cart_plus').click(function () {
        var Form = $('.wc_cart_add[id="' + $(this).attr('id') + '"]');
        var Input = Form.find('input[name="item_amount"]');
        var Amount = parseInt(Input.val()) + parseInt(1);

        Form.find('.trigger').remove();

        if (parseInt(Amount) <= parseInt(Input.attr('max')) || !Input.attr('max')) {
            Input.val(Amount);
        } else {
            Input.val(Input.attr('max'));
            Form.find('button').last().after("<p class='m_top ds_none trigger trigger_info none'><b>OPPSSS:</b> Temos " + Input.attr('max') + " unidades em estoque!");
            Form.find('.trigger').fadeIn(400);
        }
        
        wcZipRecalculate(); // LINHA NOVA
        return false;
    });

    //LESS ITEM AMOUNT
    $('.wc_cart_less').click(function () {
        var Form = $('.wc_cart_add[id="' + $(this).attr('id') + '"]');
        var Input = Form.find('input[name="item_amount"]');
        var Amount = parseInt(Input.val()) - parseInt(1);

        Form.find('.trigger').fadeOut(1, function () {
            $(this).remove();
        });

        if (parseInt(Amount) >= 1) {
            Input.val(Amount);
        }
        
        wcZipRecalculate(); // LINHA NOVA
        return false;
    });
```

#### 3 - Altere o arquivo "_cdn/widgets/ecommerce/cat.ajax.php"
##### - dentro do "case 'cart_shipment'", substituir TODO o foreach da linha 155 até 168, pelo código abaixo:
```php
    // calculo do frete sem produto no cart
    if (empty($_SESSION['wc_order']) || isset($POST['ItemAmount'])):
        $ItemId = $POST['pdtId'];
        $ItemAmount = $POST['ItemAmount'];
        $Read->ExeRead(DB_PDT, "WHERE pdt_id = :id", "id={$ItemId}");
        if (!$Read->getResult()):
            unset($_SESSION['wc_order'][$ItemId]);
        else:
            extract($Read->getResult()[0]);
            $CartTotal += ($pdt_offer_price && $pdt_offer_start <= date('Y-m-d H:i:s') && $pdt_offer_end >= date('Y-m-d H:i:s') ? $pdt_offer_price : $pdt_price) * $ItemAmount;
            $HeightTotal += $pdt_dimension_heigth * $ItemAmount;
            $WidthTotal += $pdt_dimension_width * $ItemAmount;
            $DepthTotal += $pdt_dimension_depth * $ItemAmount;
            $WeightTotal += $pdt_dimension_weight * $ItemAmount;
            $AmountTotal += $ItemAmount;
        endif;
    else:
        foreach ($_SESSION['wc_order'] as $ItemId => $ItemAmount):
            $Read->ExeRead(DB_PDT, "WHERE pdt_id = (SELECT pdt_id FROM " . DB_PDT_STOCK . " WHERE stock_id = :id)", "id={$ItemId}");
            if (!$Read->getResult()):
                unset($_SESSION['wc_order'][$ItemId]);
            else:
                extract($Read->getResult()[0]);
                $CartTotal += ($pdt_offer_price && $pdt_offer_start <= date('Y-m-d H:i:s') && $pdt_offer_end >= date('Y-m-d H:i:s') ? $pdt_offer_price : $pdt_price) * $ItemAmount;
                $HeightTotal += $pdt_dimension_heigth * $ItemAmount;
                $WidthTotal += $pdt_dimension_width * $ItemAmount;
                $DepthTotal += $pdt_dimension_depth * $ItemAmount;
                $WeightTotal += $pdt_dimension_weight * $ItemAmount;
                $AmountTotal += $ItemAmount;
            endif;
        endforeach;
    endif;
```

##### - Procure o case "case 'cart_shipment_select':" e troque todo o foreach pelo código abaixo:
```php
    // calculo do frete sem produto no cart
    if (empty($_SESSION['wc_order']) || isset($POST['itemAmount'])):
        $ItemId = $POST['pdtId'];
        $ItemAmount = $POST['itemAmount'];

        $Read->FullRead("SELECT pdt_price, pdt_offer_price, pdt_offer_start, pdt_offer_end FROM " . DB_PDT . " WHERE pdt_id = (SELECT pdt_id FROM " . DB_PDT_STOCK . " WHERE stock_id = :id)", "id={$ItemId}");
        if (!$Read->getResult()):
            unset($_SESSION['wc_order'][$ItemId]);
        else:
            extract($Read->getResult()[0]);
            $CartTotal += ($pdt_offer_price && $pdt_offer_start <= date('Y-m-d H:i:s') && $pdt_offer_end >= date('Y-m-d H:i:s') ? $pdt_offer_price : $pdt_price) * $ItemAmount;
        endif;
    else:
        foreach ($_SESSION['wc_order'] as $ItemId => $ItemAmount):
            $Read->FullRead("SELECT pdt_price, pdt_offer_price, pdt_offer_start, pdt_offer_end FROM " . DB_PDT . " WHERE pdt_id = (SELECT pdt_id FROM " . DB_PDT_STOCK . " WHERE stock_id = :id)", "id={$ItemId}");
            if (!$Read->getResult()):
                unset($_SESSION['wc_order'][$ItemId]);
            else:
                extract($Read->getResult()[0]);
                $CartTotal += ($pdt_offer_price && $pdt_offer_start <= date('Y-m-d H:i:s') && $pdt_offer_end >= date('Y-m-d H:i:s') ? $pdt_offer_price : $pdt_price) * $ItemAmount;
            endif;
        endforeach;
    endif;
```
    
