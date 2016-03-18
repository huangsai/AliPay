# AliPay
###1. 自定义支付方法
  public void pay(final Activity activity, String platform, final Bundle args, final IPayResult listener) {

		Runnable payRunnable = new Runnable() {

			@Override
			public void run() {
				final float price = args.getFloat("price");
				// 构造PayTask 对象
				final PayTask alipay = new PayTask(activity);

				// 从服务端获取支付参数
				final Result<AlipayInfo> alipayResult = mHttpManager.requestAlipay(price);
				Log.d(TAG, alipayResult.getObject().order_param);

				// 调用支付接口，获取支付结果
				final String result = alipay.pay(alipayResult.getObject().order_param, true);
				Log.d(TAG, result);
				final PayResult payResult = new PayResult(result);

				// 支付结果回调
				activity.runOnUiThread(new Runnable() {

					@Override
					public void run() {
						if (listener != null) {
							listener.onPayResult("alipay", payResult.getResultStatus());
						}
					}
				});

			}
		};

		// 必须异步调用
		Thread payThread = new Thread(payRunnable);
		payThread.start();
	}
###2.根据服务器返回的参数 调用支付宝接口支付
      // 调用支付接口，获取支付结果
		final String result = alipay.pay(alipayResult.getObject().order_param, true);
		Log.d(TAG, result);
		final PayResult payResult = new PayResult(result);

		// 支付结果回调
		activity.runOnUiThread(new Runnable() {

			@Override
			public void run() {
				if (listener != null) {
					listener.onPayResult("alipay", payResult.getResultStatus());
				}
			}
		});
###3.自己定义一个接口 用来支付结果回调的
    public static interface IPayResult {
	    public void onPayResult(String platform, String resultCode);
  	}
  //	将支付结果解析
    public class PayResult {
    	private String resultStatus;
    	private String result;
    	private String memo;
    
    	public PayResult(String rawResult) {
    
    		if (TextUtils.isEmpty(rawResult))
    			return;
    
    		String[] resultParams = rawResult.split(";");
    		for (String resultParam : resultParams) {
    			if (resultParam.startsWith("resultStatus")) {
    				resultStatus = gatValue(resultParam, "resultStatus");
    			}
    			if (resultParam.startsWith("result")) {
    				result = gatValue(resultParam, "result");
    			}
    			if (resultParam.startsWith("memo")) {
    				memo = gatValue(resultParam, "memo");
    			}
    		}
    	}
    
    	@Override
    	public String toString() {
    		return "resultStatus={" + resultStatus + "};memo={" + memo
    				+ "};result={" + result + "}";
    	}
    
    	private String gatValue(String content, String key) {
    		String prefix = key + "={";
    		return content.substring(content.indexOf(prefix) + prefix.length(),
    				content.lastIndexOf("}"));
    	}
    
    	/**
    	 * @return the resultStatus
    	 */
    	public String getResultStatus() {
    		return resultStatus;
    	}
    
    	/**
    	 * @return the memo
    	 */
    	public String getMemo() {
    		return memo;
    	}
    
    	/**
    	 * @return the result
    	 */
    	public String getResult() {
    		return result;
    	}
}
###4.在支付页面实现接口 重写下面这个方法
	@Override
	public void onPayResult(String platform, String resultCode) {
	  if (TextUtils.equals(resultCode, "9000")) {
	  		ActivityUtils.showToast(this, "支付成功");
	  } else {
	  // 判断resultStatus 为非"9000"则代表可能支付失败
	  // "8000"代表支付结果因为支付渠道原因或者系统原因还在等待支付结果确认，最终交易是否成功以服务端异步通知为准（小概率状态）
	  	if (TextUtils.equals(resultCode, "8000")) {
	  		ActivityUtils.showToast(this, "支付结果确认中");
	
	  	} else {
	  		// 其他值就可以判断为支付失败，包括用户主动取消支付，或者系统返回的错误
	  		ActivityUtils.showToast(this, "支付失败");
	
	  	}
	  }
	}
