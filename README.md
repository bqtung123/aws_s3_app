```
class Admin::CouponsController < Admin::ApplicationController
  before_action :get_company
  before_action :set_coupon, only: %i(edit show update destroy complete
                                      get_product_list update_product_coupon
                                      user_coupon_use_histories)
  before_action :check_min_purchase_amount_int, only: :confirm
  before_action :check_discount_value_int, only: :confirm

  def index
    authorize! :read, Coupon

    @q       = @company.coupons.ransack(params[:q])
    @coupons = @q.result.paginate(page: params[:page], per_page: params[:per_page] || default_page_size)
    @coupons = @coupons.order(created_at: :desc) if @q.sorts.empty?

    @count = @coupons.count
  end

  def new
    authorize! :new, Coupon
    @coupon            = @company.coupons.new
    @coupon.attributes = coupon_params if params[:is_back]
  end

  def create
    authorize! :create, Coupon
    blob   = ActiveStorage::Blob.find_by(key: params[:url_key])
    coupon = @company.coupons.new coupon_params
    if coupon.save
      if blob.present?
        ActiveStorage::Attachment.create(
          name: "coupon_image", record_type: "Coupon", record_id: coupon.id, blob_id: blob.id
        )
      end
      redirect_to(complete_admin_coupon_path(coupon))
    else
      render(:new)
    end
  end

  def confirm
    if params[:coupon][:id].present?
      @coupon             = Coupon.find_by(id: params[:coupon][:id])
      @current_image      = Coupon.find_by(id: params[:coupon][:id]).coupon_image
      @coupon.attributes  = coupon_params
      @check_delete_image = params[:delete_upload_image]
      @coupon_image       = coupon_params[:coupon_image]
      authorize! :update, @coupon
      render :edit if @coupon.invalid? || @min_purchase_amount_is_not_int || @discount_value_is_not_int
    else
      authorize! :new, Coupon
      @coupon = @company.coupons.new(coupon_params)
      render :new if @coupon.invalid? || @min_purchase_amount_is_not_int || @discount_value_is_not_int
    end
    @img_url = create_blob_and_get_url if coupon_params[:coupon_image].present? && @coupon.valid?
  end

  def complete
  end

  def edit
    authorize! :edit, @coupon
    @current_image     = @coupon.coupon_image
    @coupon.attributes = coupon_params if params[:is_back]
  end

  def show
    authorize! :show, @coupon
    qrcode = RQRCode::QRCode.new("https://#{@company.sub_domain}.#{Settings.domain}/coupon/qr/regist/#{@coupon.code}")
    @svg   = qrcode.as_svg(
      offset:          0,
      color:           "000",
      shape_rendering: "crispEdges",
      module_size:     3
    )
  end

  def update
    authorize! :update, @coupon
    blob = ActiveStorage::Blob.find_by(key: params[:url_key])
    if @coupon.update(coupon_params)
      handle_update_image if params[:delete_upload_image].present?
      if blob.present?
        ActiveStorage::Attachment.find_or_create_by(name: "coupon_image", record_type: "Coupon", record_id: @coupon.id)
                                 .update(blob_id: blob.id)
      end
      redirect_to(complete_admin_coupon_path(@coupon))
    else
      render(:edit)
    end
  end

  def user_coupon_use_histories
    authorize! :read, UserCouponUseHistory
    @histories  = @coupon.user_coupon_use_histories.order(created_at: :desc)
                         .paginate(page: params[:page], per_page: params[:per_page] || default_page_size)
    @used_times = @histories.pluck(:used_time)
    @count      = @histories.count
  end

  def get_product_list
    authorize! :read, Coupon

    @product_categories = ProductCategory.all
    @brands             = Brand.all

    if params[:q]
      params[:q][:product_code_or_product_name_cont] = params[:q][:code_or_name_cont]
      if params[:q][:s] || params[:page]
        params[:q][:product_category_name_matches_any] = params[:q][:product_category_name_matches_any].join("  ") rescue nil
        params[:q][:product_brand_name_matches_any]    = params[:q][:brand_name_matches_any].join("  ") rescue nil
      end
    end

    @product_category_names = handle_q_matches_all(:product_category_id_matches_any, :product_category_id_matches_any)
    @brand_names            = handle_q_matches_all(:product_brand_id_matches_any, :brand_id_matches_any)

    if params[:q] && params[:q][:product_category_id_matches_any].present?
      params[:q][:product_category_id_or_sub_category_id_in] = params[:q][:product_category_id_matches_any]
      params[:q][:product_category_id_matches_any] = []
    end

    @code_or_name           = params[:q][:code_or_name_cont] if params[:q] && params[:q][:code_or_name_cont]
    @q                      = Selling.selling_products(@company.id).ransack(params[:q])
    @page                   = params[:page] || 1

    @products = @q.result.paginate(page: @page, per_page: params[:per_page] || default_page_size)
    @products = @products.includes(:product_coupons).order(created_at: :desc) if @q.sorts.empty?
    @count    = @products.count
  end

  def update_product_coupon
    product_coupons = @coupon.product_coupons.with_discarded
    (params[:historyCheck] || {}).each do |key, value|
      product_coupon = product_coupons.find_by(selling_id: key)
      if value == "true"
        if product_coupon.present?
          product_coupon.undiscard if product_coupon.discarded?
        else
          @coupon.product_coupons.create(selling_id: key)
        end
      elsif value == "false" && product_coupon.present?
        product_coupon.discard
      end
    end

    flash[:success] = t(".update_product_coupon_success")
  end

  def destroy
    authorize! :destroy, @coupon
    @coupon.discard
    redirect_to admin_coupons_url
  end

  private

  def coupon_params
    params.require(:coupon).permit(:name, :code, :usage_limit_flag,
                                   :usage_number, :discount_type, :discount_value, :min_purchase_amount,
                                   :auto_grant_flag, :start_date, :end_date, :coupon_image)
  end

  def handle_q_matches_all search_attribute, attribute
    if params[:q] && params[:q][attribute]
      params[:q][search_attribute] = params[:q][attribute].split("  ")
      params[:q][search_attribute]
    end
  end

  def set_coupon
    @coupon = @company.coupons.find_by(id: params[:id])
  end

  def handle_update_image
    attachment = ActiveStorage::Attachment.find_by(name: "coupon_image", record_type: "Coupon", record_id: @coupon.id).destroy
    ActiveStorage::Blob.find_by(id: attachment.blob_id).destroy if attachment
  end

  def check_min_purchase_amount_int
    min_purchase_amount             = params[:coupon][:min_purchase_amount].to_f
    @min_purchase_amount_is_not_int = min_purchase_amount != min_purchase_amount.to_i
  end

  def check_discount_value_int
    discount_value             = params[:coupon][:discount_value].to_f
    @discount_value_is_not_int = discount_value != discount_value.to_i
  end

  def create_blob_and_get_url
    image_params = coupon_params[:coupon_image]
    image        = ActiveStorage::Blob.create_and_upload!(io: image_params, filename: image_params.original_filename)
    object       = S3_BUCKET.object(image.key)
    object.presigned_url(:get).to_s

    {
      key: image.key,
      url: object.presigned_url(:get).to_s
    }
  rescue StandardError
    ""
  end

  def get_company
    @company = current_admin_user.company
  end
end
```
