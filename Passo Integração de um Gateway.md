# Passo Integração de um Gateway

1. Criar a branch

```
git checkout -b feat/[nº task]- [nome gateway]
```

2. Acesse Procfile.dev

```
make run
```

```
make bash
```

```???
./bin/webpack-dev-server
```

```
bundle exec rails s -b 0.0.0.0 -p 3001
```

3. Criar o generator do gateway 

```
railg g model [Gateway]payConfig
```

4. Ir  para Migrate Referencia  *creat_armpay_configs*,  gera um copia e ajusta com a novo Gateway

        OBS:  Detalhe para a url -> **srv**

        Nos dados(trello)  copiar URL Base para configuração 

(modelo)

*classepay/db/migrate/20250114174706_create_armpay_configs.rb*

```
class CreateArmpayConfigs < ActiveRecord::Migration[7.0]
  def change
    create_table :armpay_configs do |t|
      t.boolean :active, default: true
      t.boolean :capture, default: false
      t.datetime :deleted_at
      t.boolean :has_rate, default: false
      t.integer :installments
      t.string :name
      t.string :name_invoice
      t.decimal :rate, precision: 19, scale: 2
      t.string :public_key
      t.string :secret_key
      t.string :production_url, default: 'https://srv.armpay.com.br'
      t.string :sandbox_url, default: ''
      t.string :slug
      t.string :statement_descriptor
      t.references :client, null: true, foreign_key: true
      t.references :store, null: false, foreign_key: true
      t.timestamps
    end

    add_column :clients, :has_armpay, :boolean, default: false

    Gateway.create(
      version: '1',
      current: true,
      name: 'Armpay',
      credit_card_service: 'Kalyst::Payment::CreditCard',
      bank_slip_service: 'Kalyst::Payment::Billet',
      pix_service: 'Kalyst::Payment::Pix',
      cancel_service: 'Kalyst::Payment::Cancel',
      supported_payment_kind: ['pix', 'credit_card'],
      personalized_name: 'Arm pay'
    )
  end
end
```

5. Ir no trello, e realizar download do arquivo LOGO do gateway. Enviar para o diretório:

        ***app/assets/images/gateway_images***        

6. Configurar o generator (model), copiar um modelo e ajustar os dados com o novo gateway.

        ***db/migrate/20250114174706_create_armpay_configs.rb***

(Modelo)

```
c# == Schema Information
#
# Table name: armpay_configs
#
#  id                   :bigint           not null, primary key
#  active               :boolean          default(TRUE)
#  capture              :boolean          default(FALSE)
#  deleted_at           :datetime
#  has_rate             :boolean          default(FALSE)
#  installments         :integer
#  name                 :string
#  name_invoice         :string
#  production_url       :string           default("https://srv.armpay.com.br")
#  public_key           :string
#  rate                 :decimal(19, 2)
#  sandbox_url          :string           default("")
#  secret_key           :string
#  slug                 :string
#  statement_descriptor :string
#  created_at           :datetime         not null
#  updated_at           :datetime         not null
#  client_id            :bigint
#  store_id             :bigint           not null
#
# Indexes
#
#  index_armpay_configs_on_client_id  (client_id)
#  index_armpay_configs_on_store_id   (store_id)
#
# Foreign Keys
#
#  fk_rails_...  (client_id => clients.id)
#  fk_rails_...  (store_id => stores.id)
#
class ArmpayConfig < ApplicationRecord
  ## CONSTANTS
  PUBLIC_KEY_FORMAT = /\Apk_[0-9a-z]+\z/.freeze
  SECRET_KEY_FORMAT = /\Ask_[0-9a-z]+\z/.freeze

  ## PARANOIA
  acts_as_paranoid

  belongs_to :client, optional: true
  belongs_to :store, optional: true

  validates :store, uniqueness: true
  validates :public_key, presence: true, format: {with: PUBLIC_KEY_FORMAT}
  validates :secret_key, presence: true, format: {with: SECRET_KEY_FORMAT}
  validate :gateway_online?, if: -> { public_key =~ PUBLIC_KEY_FORMAT && secret_key =~ SECRET_KEY_FORMAT }

  ## CONCERNS
  include FriendlyUuid

  ## CALLBACKS
  after_create :create_gateway_config
  after_destroy :destroy_gateway_config
  before_create :set_active_attribute

  def base_url = production_url

  def installments_rate
    has_rate ? installments : 1
  end

  def formatted_rate=(rate)
    self.rate = rate.to_s.gsub(',', '.').to_f
  end

  def client_side_gateway_data
    {
      is_active: active,
      secret_key: secret_key,
      js_sdk_path: 'https://assets.pagseguro.com.br/checkout-sdk-js/rc/dist/browser/pagseguro.min.js'
    }
  end

  def gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.find_by(gateway_id: gateway.id, store_id: store.id) if store.present?
  end

  def webhook_url
    (Rails.env.development? ? "http://" : "https://") +
    (Rails.env.production? ? "sistema.classepay.com.br" : ENV["POSTBACK_HOST"] || "homolog.classepay.com.br") +
    # "https://1b47-45-227-73-11.ngrok-free.app" +
    "/api/v1/webhook/armpay/payments"
  end

  private
  def destroy_gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.where(gateway_id: gateway.id, store_id: store.id).destroy_all if store.present?
  end

  def create_gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.kinds.each_key do |kind|
      next unless store.present?

      last_gateway_config = GatewayConfig.where(kind: kind.to_sym, store_id: store.id).order(priority: :asc).last
      index = last_gateway_config ? last_gateway_config.priority : 0

      GatewayConfig.create(
        priority: index + 1,
        status: :active,
        gateway_id: gateway.id,
        gatewable: store.seller,
        kind: kind.to_sym,
        store_id: store.id
      )
    end
  end

  def set_active_attribute
    return if store.nil?

    self.active = true if store.armpay_config.nil?
  end

  def gateway_online?
    return unless errors[:public_key].empty? && errors[:secret_key].empty?
    begin
      Kalyst::Auth::Verify.call(self)
    rescue Kalyst::Base::Error => e
      errors.add(:base, e.cause.message)
    end
  end
endalidates :public_key, presence: true, format: {with: PUBLIC_KEY_FORMAT}
  validates :secret_key, presence: true, format: {with: SECRET_KEY_FORMAT}
  validate :gateway_online?, if: -> { public_key =~ PUBLIC_KEY_FORMAT && secret_key =~ SECRET_KEY_FORMAT }

  ## CONCERNS
  include FriendlyUuid

  ## CALLBACKS
  after_create :create_gateway_config
  after_destroy :destroy_gateway_config
  before_create :set_active_attribute

  def base_url = production_url

  def installments_rate
    has_rate ? installments : 1
  end

  def formatted_rate=(rate)
    self.rate = rate.to_s.gsub(',', '.').to_f
  end

  def client_side_gateway_data
    {
      is_active: active,
      secret_key: secret_key,
      js_sdk_path: 'https://assets.pagseguro.com.br/checkout-sdk-js/rc/dist/browser/pagseguro.min.js'
    }
  end

  def gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.find_by(gateway_id: gateway.id, store_id: store.id) if store.present?
  end

  def webhook_url
    (Rails.env.development? ? "http://" : "https://") +
    (Rails.env.production? ? "sistema.classepay.com.br" : ENV["POSTBACK_HOST"] || "homolog.classepay.com.br") +
    # "https://1b47-45-227-73-11.ngrok-free.app" +
    "/api/v1/webhook/armpay/payments"
  end

  private
  def destroy_gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.where(gateway_id: gateway.id, store_id: store.id).destroy_all if store.present?
  end

  def create_gateway_config
    gateway = Gateway.find_by_name "Armpay"
    GatewayConfig.kinds.each_key do |kind|
      next unless store.present?

      last_gateway_config = GatewayConfig.where(kind: kind.to_sym, store_id: store.id).order(priority: :asc).last
      index = last_gateway_config ? last_gateway_config.priority : 0

      GatewayConfig.create(
        priority: index + 1,
        status: :active,
        gateway_id: gateway.id,
        gatewable: store.seller,
        kind: kind.to_sym,
        store_id: store.id
      )
    end
  end

  def set_active_attribute
    return if store.nil?

    self.active = true if store.armpay_config.nil?
  end

  def gateway_online?
    return unless errors[:public_key].empty? && errors[:secret_key].empty?
    begin
      Kalyst::Auth::Verify.call(self)
    rescue Kalyst::Base::Error => e
      errors.add(:base, e.cause.message)
    end
  end
end
```

8. Executar as migracções

```
rails db:migrate
```

9. Ajustando o form Clients

    ***app/views/admin/clients/_form.html.erb***

    (modelo)

```
   <div class="col-6">
       <%= f.label :has_armpay, 'Gateway Armpay', class: 'fw-bold mb-2'  %>
       <%= f.check_box :has_armpay, label: 'Sim', switch: true, help: 'text' %>
   </div>
```

<img title="" src="https://github.com/AngeloSouza1/Doc_Integr/blob/master/2025-01-15_15-54.png" alt="2025-01-15_15-54.png" width="652" data-align="left">

10. Atualize a aplicação

11. No navegador, acesse com login Master, para conferência do gateway  

***parceiros/Teste Pay Checkout***/***editar***


![2025-01-16_08-37](https://github.com/user-attachments/assets/9555195d-f469-43f7-9c11-c8193c1d300a)


(modelo)

12. Agora no login usuário (Danilo), conferir se o gateway está disponível
    ***integrações/gateways***

![2025-01-15_21-00](https://github.com/user-attachments/assets/df674aeb-91a2-4860-8769-de3c2f522192)


13.    Cadastro do gateway: 

    Selecione o gateway e realize o cadastro com os dados enviados 

- Chave Secreta

- Chave Pública
14. Acesse o arquivo Base.rb

***services/kalyst/base.rb***

adicione o gateway no trecho de código:

```
  INTEGRATIONS = %w(Proxypay Dorapag Justpay Guardpay Syfra Loftpay Novak Titanshub Alcateiapay Aionpay Payd Armpay).freeze
```

15. Criar o formulário do gateway
    
    ***app/views/store/gateways***

(modelo)
**_form_armpay.html.erb**

```
  <div id="form_armpay">
  <div class="armpay">
    <%= render_image_gateway 'gateway_images/gateway_armpay.png', 'bg-white p-1' %>
  </div>
  <%= bootstrap_form_for @armpay_config, url: @armpay_config.persisted? ? update_armpay_store_gateway_path(@armpay_config) : create_armpay_store_gateways_path do |f| %>
    <%#= show_errors_for(f.object) %>
    <div class="block block-rounded block-themed shadow-sm">
      <div class="block-header bg-white border-bottom">
        <h3 class="block-title font-weight-bold text-dark">
          Dados
        </h3>
      </div>
      <div class="block-content">
        <% if f.object.errors[:base].present? %>
          <div class="row">
            <div class="col-12">
              <p><%= f.object.errors[:base] %></p>
            </div>
          </div>
        <% end %>
        <div class="row">
          <div class="col-12">
            <%= f.text_field :public_key, placeholder: "pk_" %>
          </div>
        </div>
        <div class="row">
          <div class="col-12">
            <%= f.text_field :secret_key, placeholder: "sk_" %>
          </div>
        </div>
        <div class="row">
          <div class="col-6">
            <%= f.text_field :formatted_rate, class: 'inputmask', required: true, value: f.object.rate, append: '%' %>
          </div>
        </div>
      </div>
    </div>
    <div class="text-end">
      <%= f.submit "Salvar configuração", class: 'btn btn-primary', data: {disable_with: 'Salvando...'} %>
      <%= link_to t('.cancel', default: t("helpers.links.cancel")), session[:redirection_after_register].present? ? store_redirection_after_register_path : store_gateways_path, class: 'btn btn-outline-primary' %>
    </div>
  <% end %>
</div>
```

16. Inclusão das rotas

***config/routes/store_routes.rb***

- esta inclusão (modelo): 

<img src="https://github.com/user-attachments/assets/fc3e19c8-1d94-418c-a1ad-ec650268422d" title="" alt="2025-01-16_11-53.png" width="520">

- esta inclusão (modelo):

<img title="" src="https://github.com/user-attachments/assets/39ad814d-120d-4e65-bac4-bfc8281556d6" alt="2025-01-16_11-55.png" data-align="inline" width="520">


17. Criação do controller

***controllers/store/gateways_controller.rb***

(modelo)

```
 def create_armpay
    @armpay_config = ArmpayConfig.new(armpay_config_params)
    @armpay_config.store_id = @store.id
    if @armpay_config.valid?
      @store.deactivate_all_integrations!
      @armpay_config.save
      update_gateway_configs_status(@armpay_config)
      redirect_to store_gateways_url, notice: "Salvo com sucesso."
    else
      flash.now[:error] = "Chave inválida"
      render :new, status: :unprocessable_entity
    end
  end
```

18. Ajustar as traduções na view

***config/locales/pt-BR/activerecord.pt-BR.yml***

(modelo)

```
   armpay_config:
        public_key: Chave pública
        secret_key: Chave secreta
        formatted_rate: Taxa de parcelamento
```

19. Cadastre os dados no ambiente Master (front):
    *** Integracões/Gateways***
- Insira os dados do Gateway
  
  - Chave Pública
  
  - Chave Secreta
  
  - Taxa de Parcelamento



![2025-01-16_12-18](https://github.com/user-attachments/assets/921dd13c-fc49-45e9-9684-fd967a9ba30b)


20 - Criação do form_config do  Gateway

***views/store/gateways/_armpay_config.html.erb***

modelo)

*** _armpay_config.html.erb***

```
  <tr class="align-middle" data-armpay-id="<%= armpay.id -%>">
  <td class="text-center align-middle">
    <%= image_tag 'gateway_images/gateway_armpay.png', alt: 'armpay', height: height_image, width: width_image , class: 'object-fit-contain p-1 rounded bg-white' %>
  </td>
  <td nowrap class="text-center">
    <%= link_to armpay.active ? "Ativo" : "Inativo", 
      armpay.active ? toggle_store_gateway_path(armpay, kind: 'armpay') : active_armpay_store_gateway_path(armpay),
      method: :get,
      class: "text-white tippy fw-bold btn btn-sm btn-#{armpay.active? ? 'success' : 'danger'}",
      data: { confirm: t("helpers.confirm"),
              tippy_content: "#{armpay.active? ? 'Desativar armpay' : 'Ativar armpay'}",
              id: armpay.id} %>
</td>
<td width="200" class="options">
  <% if armpay.active? %>
    <%= link_to toggle_store_gateway_path(armpay, kind: "armpay"), class: 'tippy btn btn-sm btn-danger text-white',
      data: { confirm: 'Qualquer outra configuração ficará inativa. Continuar?', 'tippy-content': "Desativar" } do %>
      <i data-feather="x-circle"></i>
    <% end %>
  <% else %>
    <%= link_to active_armpay_store_gateway_path(armpay), class: 'tippy btn btn-sm btn-success text-white',
      data: { confirm: 'Qualquer outra configuração ficará inativa. Continuar?', 'tippy-content': t('.active', default: t("helpers.links.active")) } do %>
      <i data-feather="check-square"></i>
    <% end %>
  <% end %>
  <%= link_to edit_armpay_store_gateway_path(armpay), class: 'tippy btn btn-sm btn-primary',
    :'data-tippy-content' => t('.edit', default: t("helpers.links.edit")) do %>
    <i data-feather="edit"></i>
  <% end %>
  <%= link_to destroy_armpay_store_gateway_path(armpay), method: :delete, data: { confirm: t("helpers.confirm") }, class: 'tippy btn btn-sm btn-danger text-white',
    :'data-tippy-content' => t('.destroy', default: t("helpers.links.destroy")) do %>
    <i data-feather="trash-2"></i>
  <% end %>
</td>
</tr>
```

21 - Agora no login usuário (Danilo), conferir se o gateway está ativo

***Integrações/Gateways***

![2025-01-15_21-00](https://github.com/user-attachments/assets/b2e47ed3-6e21-4a5e-be08-0c370583db13)

22. Atualização dos arquivos à seguir: 

charge.rb (***app/models/charge.rb***)


![2025-01-16_13-15](https://github.com/user-attachments/assets/60529c1d-f627-42d2-bea3-3cbddbf467d9)


external_reference.rb(***models/external_reference.rb***)


![2025-01-16_13-25](https://github.com/user-attachments/assets/9d903443-12b2-49ba-927b-984cdbc9a4c6)


installment.rb(***app/models/installment.rb***)


![2025-01-16_13-28](https://github.com/user-attachments/assets/9080a59a-9add-4159-bea2-f4c7397adb36)


23. Crie um servidor ngrok

```
ngrok http 3001
```

exemplo:

***https://feaf-45-227-73-84.ngrok-free.app***

24. N arquivo config em ***app/models***, confirme esta alteração: 

<img title="" src="https://github.com/user-attachments/assets/bd2e17ea-6545-48d9-a7be-17705153092d" alt="2025-01-16_13-46.png" width="882">



25. Testar o gateway com um produto, e observar se esta ativo, no ambiente usuario,
- Produto Y, copie seu link, para verificação no navegador

(imagem demonstrativa)

![2025-01-16_13-09](https://github.com/user-attachments/assets/90521ec8-d8ce-4ad9-94d8-df999d290a3c)



- Insira dados bancarios para testes : Pix e Cartão
26. Após inserção dos dados testar o pagamento de Pix ou Cartão

![2025-01-16_13-50](https://github.com/user-attachments/assets/3886d226-b719-4976-9d73-ec7a1ff35c72)


27. Confirma os logs (***log/kalyst.log***)
    
    (imagem ilustrativa)
    
![2025-01-16_13-58](https://github.com/user-attachments/assets/4033610b-2f5b-4815-939d-d4be02749ea4)



29. Ajuste as rotas

****config/routes/api_routes.rb****

(modelo)

```
 namespace :armpay do
              get 'payments', to: 'payments#update'
              post 'payments', to: 'payments#update'
            end
```

29. Ajuste do controllers

***controllers/api/v1/webhook***

(modelo)

***armpay/payments.controller***

```
 class Api::V1::Webhook::Armpay::PaymentsController < ApiController
  skip_before_action :authenticate_request

  def update
    Kalyst::Webhook::Payment::UpdatedJob.send((Rails.env.production? ? :perform_later : :perform_now), payment_params)
    render json: 'ok'
  end

  private

  def payment_params
    params.permit!.to_h.merge({
      classepay: {
        kind_gateway: 'armpay'
      }
    })
  end
end
```

30. Gerar o pix e realizar pagamento para teste
