
# Как писать хороший код?
# Как тестировать код правильно?
# Как улучшать имеющийся код?

---

# *Как улучшать имеющийся код?*

---

#Рефакторинг. С чего начать?

##Иван Немытченко, @inem

###28 июня 2014, Omsk ruby developers meetup #2

---

# Проекту пять лет
# Rails 2.3

---

```ruby

  def create
    if current_user.space_quota_exceeded?
      render :json => {"success"=> false, "error"=>"storage quota exceeded"}
      return
    end

    upload_uuid = params[:qquuid]
    chunk_number = params[:qqpartindex].to_i
    chunks_total = params[:qqtotalparts].to_i
    temp_location = Rails.root + 'tmp/cache/upload-chunk-cache' + upload_uuid
    temp_filename = temp_location + sprintf("%06d", chunk_number)
    Rails.logger.info("will save chunk #{chunk_number}/#{chunks_total} to #{temp_filename}")
    mkdir_p temp_location unless temp_location.exist?
    File.open(temp_filename, 'wb') do |f|
      while (chunk = request.body.read(512.kilobytes)) != nil
        f.write chunk
      end
    end

    # rename chunk to *.ready:
    File.rename temp_filename, temp_filename.to_s+".ready"

    # check if all chunks are ready:
    if Dir.glob(temp_location + "*.ready").size == chunks_total

      user = current_user
      if(params[:started_by_id] and current_user.admin?)
        user = User.find(params[:started_by_id])
      end
      if not current_user.admin?
        # enforce non-admin rules from #72698174
        params[:nodes_number] = 1
        params[:parallelserial] = "serial"
      end

      #if auto-mode
      if(params[:workflow_id] == '')
        Rails.logger.info("Going to find project & workflow automatically")

        file_name = params[:qqfile]
        accession = Accession.new(file_name)
        sample = Maybe(Sample.find_by_accession(accession))
        projects = sample.projects
        workflows = projects.map(&:workflow).compact

        error = nil
        if sample.nil?
          error = "Can't find related sample"
        elsif projects.empty?
          error = "Can't find related project"
        elsif workflows.empty?
          error = "There's no workflows in all of the related projects"
        end

        if error
          puts "Oops. #{file_name} will not start search: #{error}"
          Rails.logger.info("Oops. #{file_name} will not start search: #{error}")

          Notification.create(:user => user, :message => "#{error} for uploaded file #{file_name}")
          render :json => { "success"=> false, "error" => error } and return
        end

        search_parameters = workflows.first.search_parameters
        workflow_id = nil
        sample_id = sample.id

        #manual mode
      else
        workflow_id = params[:workflow_id]
        sample_id = nil

        if workflow_id
          search_parameters = Workflow.find(params[:workflow_id]).search_parameters
          # It seems it only makes sence to do this verification for regular uploads, not for Auto-mode
          cluster_options = {:runs => [ 1, 2, 3 ],  # no real runs, but we can check with any
                :number_of_instances => params[:nodes_number].to_i,
                :search_parameters => search_parameters,
                :search_owner => user,
                :started_by => current_user,
                :parallelserial => params[:parallel_serial],
                :separately => true}

          can, error = Search.check_if_can_start_searches_cluster(cluster_options)
          if can
            Rails.logger.info("Yes, this can start")
          else
            puts "Oops. #{file_name} will not start search: #{error}"
            Rails.logger.info("Oops. #{file_name} will not start search: #{error}")
            render :json => { "success"=> false, "error" => error } and return
          end
        end

        # find a project to attach this upload:
        proj = nil
        if(params[:project_id])
          if(params[:project_id] == "none")
            name = params[:new_project_name]
            Rails.logger.info("Will create new project #{name}")
            # user requested project creation, but first check if he already has project with this name:
            proj = Project.find(:first, :conditions => {:name => name, :user_id => user.id}) or
            proj = Project.create(:name => name, :user => user) if proj.nil?
          else
            proj = Project.find(params[:project_id])
            unless proj.user == current_user or current_user.admin?
              proj = nil
            end
          end
        end
        Rails.logger.info("Will add to project project ##{proj.id} #{proj.name}") unless proj.nil?
      end


      # create and enqueue RunUpload:
      run_upload = RunUpload.create(:original_name => params[:qqfile],
                                    :upload_ip => request.remote_ip,
                                    :user => proj ? proj.user : current_user,
                                    :started_by => current_user,
                                    :workflow_id => workflow_id,
                                    :parallel_serial => params[:parallel_serial],
                                    :number_of_nodes => params[:nodes_number],
                                    :project => proj,
                                    :status => 'uploading',
                                    :sample_id => sample_id)

      # move all cached chunks into run_upload dir:
      upload_dir = run_upload.local_name.dirname
      FileUtils.mv temp_location, upload_dir, :verbose => true

      # enqueue RunUpload
      if run_upload.valid?
        if RAILS_ENV == 'development'
          RunUpload.perform(run_upload.id)
        else
          Resque.enqueue(RunUpload, run_upload.id)
          run_upload.update_attribute :status, 'pending'
        end

        render :content_type => 'text/html', :json => {"success"=> true}
      else
        run_upload.update_attribute :status => 'error'
        ActiveRecord::Base.logger.error(run_upload.errors.inspect)
        render :content_type => 'text/html', :json => {"success"=> false, "error"=>"Error occured. Check server log files for details"}
      end
    else
      # this was not the last chunk, so report success:
      render :content_type => 'text/html', :json => {"success"=> true}
    end
  end

```

---

# Если подходить академически, то кажется что плохо все

![Right](poos.jpg)

---

# 1. Но оно работает
# 2. Заказчик хочет продолжения банкета

---

#С чего начать?

---

#Don't do it for free

---

#Don't do it for free

![](time-is-money.jpg)

---

#Don't push it too hard.

---

#Don't push it too hard.

![](austronaut0.jpg)

---

#Контроллерам - контроллерово!


```ruby
  def create
    use_case = UseCases::Samples::Create.new(current_user, @project.id)
    @samples = use_case.run(params[:samples])
    render_results
  end

  def mass_update
    use_case = UseCases::Samples::Update.new(current_user, @project.id)
    use_case.run(params[:samples])
    render_results
  end
```

---

#Бизнес-логику - юз-кейсам!


![Fit](use_cases_structure.png)

---

```ruby
  module UseCases::Samples
    class UseCase
      def initialize(initiator, project_id)
        @initiator = initiator
        @project_id = project_id
      end

      def run(input_data)
        some_really_complex_stuff_here do |data|
          b = bla(data)
          c = blabla(project_id, data, b)
          bla!(initiator, project_id, c)
        end
      end

      private
      attr_accessor :initiator, :project_id

      def project
        @project ||= Project.find_by_id(project_id)
      end
    end
  end
```

---

#Можно ли вызывать юз-кейс из другого?

---
# Вопросы?

![](austronaut.jpg)