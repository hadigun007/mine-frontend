<center><h1>GoAdmin References</h1></center>

## [Controller](#controllers) | [Database](#databases)

### Controllers
- [ApiCreate](#api_creatego--apicreate)
- [ApiCreateForm](#api_creatego--apicreateform)
- [ApiDetail](#api_detailgo--apidetail)
- [ApiList](#api_listgo--apilist)
- [ApiUpdate](#api_updatego--apiupdate)
- [ApiUpdateForm](#api_updatego--apiupdateform)
- [Auth](#authgo--auth)
- [Logout](#authgo--logout)
- [ShowLogin](#authgo--showlogin)
- [Delete](#deletego--delete)
- [ShowDetail](#detailgo--detail)
- [ShowForm](#editgo--showform)
- [EditForm](#editgo--editform)
- [GlobalDetaultHandler](#handlergo--globaldefenderhandler)
- [ShowInstall](#installgo--showinstall)
- [CheckDatabase](#installgo--checkdatabase)
- [ShowMenu](#menugo--showmenu)
- [ShowNewMenu](#menugo--showmenu)
- [ShowEditMenu](#menugo--showeditmenu)
- [DeleteMenu](#menugo--deletemenu)
- [EditMenu](#menugo--editmenu)
- [NewMenu](#menugo--newmenu)
- [MenuOrder](#menugo--menuorder)
- [ShowNewForm](#newgo--shownewform)
- [NewForm](#newgo--newform)
- [Operation](#operationgo--operation)
- [RecordOperationLog](#operationgo--recordoperationlog)
- [GetPluginsPageJS](#plugins_tmplgo--getpluginpagejs)
- [Plugins](#pluginsgo--plugins)
- [PluginStore](#pluginsgo--pluginstore)
- [PluginDetail](#pluginsgo--plugindetail)
- [GetPluginBoxParamFromPlug](#pluginsgo--getpluginparamfromplug)
- [PluginDownload](#pluginsgo--plugindownload)
- [ServerLogin](#pluginsgo--serverlogin)
- [ShowInfo](#showgo--showinfo)
- [Assets](#showgo--assets)
- [Export](#showgo--export)
- [SystemInfo](#systemgo--systeminfo)
- [Update](#updatego--update)
- [TestInfoUrl](#common_testgo--testisinfourl)
- [TestIsNewUrl](#common_testgo--testisnewurl)

-----------------------------------
### Controller API

##### api_create.go > ApiCreate
``` shell
# Learning
func (h *Handler) ApiCreate(ctx *context.Context) {
	param := guard.GetNewFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	err := param.Panel.InsertData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}
```

##### api_create.go > ApiCreateForm
``` shell
# Learning
func (h *Handler) ApiCreateForm(ctx *context.Context) {

	var (
		params           = guard.GetShowNewFormParam(ctx)
		prefix, paramStr = params.Prefix, params.Param.GetRouteParamStr()
		panel            = h.table(prefix, ctx)
		formInfo         = panel.GetNewFormInfo()
		infoUrl          = h.routePathWithPrefix("api_info", prefix) + paramStr
		newUrl           = h.routePathWithPrefix("api_new", prefix)
		referer          = ctx.Referer()
		f                = panel.GetActualNewForm()
	)

	if referer != "" && !isInfoUrl(referer) && !isNewUrl(referer, ctx.Query(constant.PrefixKey)) {
		infoUrl = referer
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"new":  newUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter("new", f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormNewBtnWord),
	})
}
```

##### api_detail.go > ApiDetail
``` shell
# Learning
func (h *Handler) ApiDetail(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)
	id := ctx.Query(constant.DetailPKKey)
	panel := h.table(prefix, ctx)
	user := auth.Auth(ctx)

	newPanel := panel.Copy()

	formModel := newPanel.GetForm()

	var fieldList types.FieldList

	if len(panel.GetDetail().FieldList) == 0 {
		fieldList = panel.GetInfo().FieldList
	} else {
		fieldList = panel.GetDetail().FieldList
	}

	formModel.FieldList = make([]types.FormField, len(fieldList))

	for i, field := range fieldList {
		formModel.FieldList[i] = types.FormField{
			Field:        field.Field,
			FieldClass:   field.Field,
			TypeName:     field.TypeName,
			Head:         field.Head,
			Hide:         field.Hide,
			Joins:        field.Joins,
			FormType:     form.Default,
			FieldDisplay: field.FieldDisplay,
		}
	}

	param := parameter.GetParam(ctx.Request.URL,
		panel.GetInfo().DefaultPageSize,
		panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	paramStr := param.DeleteDetailPk().GetRouteParamStr()

	editUrl := modules.AorEmpty(!panel.GetInfo().IsHideEditButton, h.routePathWithPrefix("show_edit", prefix)+paramStr+
		"&"+constant.EditPKKey+"="+ctx.Query(constant.DetailPKKey))
	deleteUrl := modules.AorEmpty(!panel.GetInfo().IsHideDeleteButton, h.routePathWithPrefix("delete", prefix)+paramStr)
	infoUrl := h.routePathWithPrefix("info", prefix) + paramStr

	editUrl = user.GetCheckPermissionByUrlMethod(editUrl, h.route("show_edit").Method())
	deleteUrl = user.GetCheckPermissionByUrlMethod(deleteUrl, h.route("delete").Method())

	deleteJs := ""

	if deleteUrl != "" {
		deleteJs = fmt.Sprintf(`<script>
function DeletePost(id) {
	swal({
			title: '%s',
			type: "warning",
			showCancelButton: true,
			confirmButtonColor: "#DD6B55",
			confirmButtonText: '%s',
			closeOnConfirm: false,
			cancelButtonText: '%s',
		},
		function () {
			$.ajax({
				method: 'post',
				url: '%s',
				data: {
					id: id
				},
				success: function (data) {
					if (typeof (data) === "string") {
						data = JSON.parse(data);
					}
					if (data.code === 200) {
						location.href = '%s'
					} else {
						swal(data.msg, '', 'error');
					}
				}
			});
		});
}

$('.delete-btn').on('click', function (event) {
	DeletePost(%s)
});

</script>`, language.Get("are you sure to delete"), language.Get("yes"), language.Get("cancel"), deleteUrl, infoUrl, id)
	}

	desc := panel.GetDetail().Description

	if desc == "" {
		desc = panel.GetInfo().Description + language.Get("Detail")
	}

	formInfo, err := newPanel.GetDataWithId(param.WithPKs(id))

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":    formInfo,
		"previous": infoUrl,
		"footer":   deleteJs,
		"prefix":   h.config.PrefixFixSlash(),
	})
}

```

##### api_list.go > ApiList
``` shell
# Learning
func (h *Handler) ApiList(ctx *context.Context) {
	prefix := ctx.Query(constant.PrefixKey)

	panel := h.table(prefix, ctx)

	params := parameter.GetParam(ctx.Request.URL, panel.GetInfo().DefaultPageSize, panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	panel, panelInfo, urls, err := h.showTableData(ctx, prefix, params, panel, "api_")
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"panel":  panelInfo,
		"footer": panelInfo.Paginator.GetContent() + panel.GetInfo().FooterHtml,
		"header": aDataTable().GetDataTableHeader() + panel.GetInfo().HeaderHtml,
		"prefix": h.config.PrefixFixSlash(),
		"urls": map[string]string{
			"edit":   urls[0],
			"new":    urls[1],
			"delete": urls[2],
			"export": urls[3],
			"detail": urls[4],
			"info":   urls[5],
			"update": urls[6],
		},
	})
}
```

##### api_update.go > ApiUpdate
``` shell
# Learning
func (h *Handler) ApiUpdate(ctx *context.Context) {
	param := guard.GetEditFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			response.Error(ctx, err.Error())
			return
		}
	}

	for i := 0; i < len(param.Panel.GetForm().FieldList); i++ {
		if param.Panel.GetForm().FieldList[i].FormType == form.File &&
			len(param.MultiForm.File[param.Panel.GetForm().FieldList[i].Field]) == 0 &&
			param.MultiForm.Value[param.Panel.GetForm().FieldList[i].Field+"__delete_flag"][0] != "1" {
			delete(param.MultiForm.Value, param.Panel.GetForm().FieldList[i].Field)
		}
	}

	err := param.Panel.UpdateData(param.Value())
	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}

```

##### api_update.go > ApiUpdateForm
``` shell
# Learning
func (h *Handler) ApiUpdateForm(ctx *context.Context) {
	params := guard.GetShowFormParam(ctx)

	prefix, param := params.Prefix, params.Param

	panel := h.table(prefix, ctx)

	user := auth.Auth(ctx)

	paramStr := param.GetRouteParamStr()

	newUrl := modules.AorEmpty(panel.GetCanAdd(), h.routePathWithPrefix("api_show_new", prefix)+paramStr)
	footerKind := "edit"
	if newUrl == "" || !user.CheckPermissionByUrlMethod(newUrl, h.route("api_show_new").Method(), url.Values{}) {
		footerKind = "edit_only"
	}

	formInfo, err := panel.GetDataWithId(param)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	infoUrl := h.routePathWithPrefix("api_info", prefix) + param.DeleteField(constant.EditPKKey).GetRouteParamStr()
	editUrl := h.routePathWithPrefix("api_edit", prefix)

	f := panel.GetForm()

	response.OkWithData(ctx, map[string]interface{}{
		"panel": formInfo,
		"urls": map[string]string{
			"info": infoUrl,
			"edit": editUrl,
		},
		"pk":     panel.GetPrimaryKey().Name,
		"header": f.HeaderHtml,
		"footer": f.FooterHtml,
		"prefix": h.config.PrefixFixSlash(),
		"token":  h.authSrv().AddToken(),
		"operation_footer": formFooter(footerKind, f.IsHideContinueEditCheckBox, f.IsHideContinueNewCheckBox,
			f.IsHideResetButton, f.FormEditBtnWord),
	})
}
```

##### auth.go > Auth 
``` shell
# Auth check input passsword and username for authentication.
func (h *Handler) Auth(ctx *context.Context) {

	var (
		user     models.UserModel
		ok       bool
		errMsg   = "fail"
		s, exist = h.services.GetOrNot(auth.ServiceKey)
	)

	if capDriver, ok := h.captchaConfig["driver"]; ok {
		capt, ok := captcha.Get(capDriver)

		if ok {
			if !capt.Validate(ctx.FormValue("token")) {
				response.BadRequest(ctx, "wrong captcha")
				return
			}
		}
	}

	if !exist {
		password := ctx.FormValue("password")
		username := ctx.FormValue("username")

		if password == "" || username == "" {
			response.BadRequest(ctx, "wrong password or username")
			return
		}
		user, ok = auth.Check(password, username, h.conn)
	} else {
		user, ok, errMsg = auth.GetService(s).P(ctx)
	}

	if !ok {
		response.BadRequest(ctx, errMsg)
		return
	}

	err := auth.SetCookie(ctx, user, h.conn)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	if ref := ctx.Referer(); ref != "" {
		if u, err := url.Parse(ref); err == nil {
			v := u.Query()
			if r := v.Get("ref"); r != "" {
				rr, _ := url.QueryUnescape(r)
				response.OkWithData(ctx, map[string]interface{}{
					"url": rr,
				})
				return
			}
		}
	}

	response.OkWithData(ctx, map[string]interface{}{
		"url": h.config.GetIndexURL(),
	})
}
```

##### auth.go > Logout
``` shell
# Logout delete the cookie
func (h *Handler) Logout(ctx *context.Context) {
	err := auth.DelCookie(ctx, db.GetConnection(h.services))
	if err != nil {
		logger.Error("logout error", err)
	}
	ctx.AddHeader("Location", h.config.Url(config.GetLoginUrl()))
	ctx.SetStatusCode(302)
}
```

##### auth.go > ShowLogin
``` shell
# ShowLogin show the login page.
func (h *Handler) ShowLogin(ctx *context.Context) {

	tmpl, name := template.GetComp("login").GetTemplate()
	buf := new(bytes.Buffer)
	if err := tmpl.ExecuteTemplate(buf, name, struct {
		UrlPrefix string
		Title     string
		Logo      template2.HTML
		CdnUrl    string
		System    types.SystemInfo
	}{
		UrlPrefix: h.config.AssertPrefix(),
		Title:     h.config.LoginTitle,
		Logo:      h.config.LoginLogo,
		System: types.SystemInfo{
			Version: system.Version(),
		},
		CdnUrl: h.config.AssetUrl,
	}); err == nil {
		ctx.HTML(http.StatusOK, buf.String())
	} else {
		logger.Error(err)
		ctx.HTML(http.StatusOK, "parse template error (；′⌒`)")
	}
}
```

##### delete.go > Delete 
``` shell
# Delete delete the row from database
func (h *Handler) Delete(ctx *context.Context) {

	param := guard.GetDeleteParam(ctx)

	#token := ctx.FormValue("_t")
	#
	#if !auth.TokenHelper.CheckToken(token) {
	#	ctx.SetStatusCode(http.StatusBadRequest)
	#	ctx.WriteString(`{"code":400, "msg":"delete fail"}`)
	#	return
	#}

	if err := h.table(param.Prefix, ctx).DeleteData(param.Id); err != nil {
		logger.Error(err)
		response.Error(ctx, "delete fail")
		return
	}

	response.OkWithData(ctx, map[string]interface{}{
		"token": h.authSrv().AddToken(),
	})
}
```

##### detail.go > Detail
``` shell
# ..
func (h *Handler) ShowDetail(ctx *context.Context) {

	var (
		prefix    = ctx.Query(constant.PrefixKey)
		id        = ctx.Query(constant.DetailPKKey)
		panel     = h.table(prefix, ctx)
		user      = auth.Auth(ctx)
		newPanel  = panel.Copy()
		detail    = panel.GetDetail()
		info      = panel.GetInfo()
		formModel = newPanel.GetForm()
		fieldList = make(types.FieldList, 0)
	)

	if len(detail.FieldList) == 0 {
		fieldList = info.FieldList
	} else {
		fieldList = detail.FieldList
	}

	formModel.FieldList = make([]types.FormField, len(fieldList))

	for i, field := range fieldList {
		formModel.FieldList[i] = types.FormField{
			Field:        field.Field,
			FieldClass:   field.Field,
			TypeName:     field.TypeName,
			Head:         field.Head,
			Hide:         field.Hide,
			Joins:        field.Joins,
			FormType:     form.Default,
			FieldDisplay: field.FieldDisplay,
		}
	}

	if detail.Table != "" {
		formModel.Table = detail.Table
	} else {
		formModel.Table = info.Table
	}

	param := parameter.GetParam(ctx.Request.URL,
		info.DefaultPageSize,
		info.SortField,
		info.GetSort())

	paramStr := param.DeleteDetailPk().GetRouteParamStr()

	editUrl := modules.AorEmpty(!info.IsHideEditButton, h.routePathWithPrefix("show_edit", prefix)+paramStr+
		"&"+constant.EditPKKey+"="+ctx.Query(constant.DetailPKKey))
	deleteUrl := modules.AorEmpty(!info.IsHideDeleteButton, h.routePathWithPrefix("delete", prefix)+paramStr)
	infoUrl := h.routePathWithPrefix("info", prefix) + paramStr

	editUrl = user.GetCheckPermissionByUrlMethod(editUrl, h.route("show_edit").Method())
	deleteUrl = user.GetCheckPermissionByUrlMethod(deleteUrl, h.route("delete").Method())

	deleteJs := ""

	if deleteUrl != "" {
		deleteJs = fmt.Sprintf(`<script>
function DeletePost(id) {
	swal({
			title: '%s',
			type: "warning",
			showCancelButton: true,
			confirmButtonColor: "#DD6B55",
			confirmButtonText: '%s',
			closeOnConfirm: false,
			cancelButtonText: '%s',
		},
		function () {
			$.ajax({
				method: 'post',
				url: '%s',
				data: {
					id: id
				},
				success: function (data) {
					if (typeof (data) === "string") {
						data = JSON.parse(data);
					}
					if (data.code === 200) {
						location.href = '%s'
					} else {
						swal(data.msg, '', 'error');
					}
				}
			});
		});
}

$('.delete-btn').on('click', function (event) {
	DeletePost(%s)
});

</script>`, language.Get("are you sure to delete"), language.Get("yes"),
			language.Get("cancel"), deleteUrl, infoUrl, id)
	}

	title := ""
	desc := ""

	isNotIframe := ctx.Query(constant.IframeKey) != "true"

	if isNotIframe {
		title = detail.Title

		if title == "" {
			title = info.Title + language.Get("Detail")
		}

		desc = detail.Description

		if desc == "" {
			desc = info.Description + language.Get("Detail")
		}
	}

	formInfo, err := newPanel.GetDataWithId(param.WithPKs(id))

	if err != nil {
		h.HTML(ctx, user, template.WarningPanelWithDescAndTitle(err.Error(), desc, title),
			template.ExecuteOptions{Animation: param.Animation})
		return
	}

	h.HTML(ctx, user, types.Panel{
		Content: detailContent(aForm().
			SetTitle(template.HTML(title)).
			SetContent(formInfo.FieldList).
			SetHeader(detail.HeaderHtml).
			SetFooter(template.HTML(deleteJs)+detail.FooterHtml).
			SetHiddenFields(map[string]string{
				form2.PreviousKey: infoUrl,
			}).
			SetPrefix(h.config.PrefixFixSlash()), editUrl, deleteUrl, !isNotIframe),
		Description: template.HTML(desc),
		Title:       template.HTML(title),
	}, template.ExecuteOptions{Animation: param.Animation})
}
```

##### edit.go > ShowForm
``` shell
# ShowFrom show form page
func (h *Handler) ShowForm(ctx *context.Context) {
	param := guard.GetShowFormParam(ctx)
	h.showForm(ctx, "", param.Prefix, param.Param, false)
}

```

##### edit.go > EditForm
``` shell
# ...
func (h *Handler) EditForm(ctx *context.Context) {

	param := guard.GetEditFormParam(ctx)

	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			logger.Error("get file engine error: ", err)
			if ctx.WantJSON() {
				response.Error(ctx, err.Error())
			} else {
				h.showForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param, true)
			}
			return
		}
	}

	formPanel := param.Panel.GetForm()

	for i := 0; i < len(formPanel.FieldList); i++ {
		if formPanel.FieldList[i].FormType == form.File &&
			len(param.MultiForm.File[formPanel.FieldList[i].Field]) == 0 &&
			len(param.MultiForm.Value[formPanel.FieldList[i].Field+"__delete_flag"]) > 0 &&
			param.MultiForm.Value[formPanel.FieldList[i].Field+"__delete_flag"][0] != "1" {
			param.MultiForm.Value[formPanel.FieldList[i].Field] = []string{""}
		}
		if formPanel.FieldList[i].FormType == form.File &&
			len(param.MultiForm.Value[formPanel.FieldList[i].Field+"__change_flag"]) > 0 &&
			param.MultiForm.Value[formPanel.FieldList[i].Field+"__change_flag"][0] != "1" {
			delete(param.MultiForm.Value, formPanel.FieldList[i].Field)
		}
	}

	err := param.Panel.UpdateData(param.Value())
	if err != nil {
		logger.Error("update data error: ", err)
		if ctx.WantJSON() {
			response.Error(ctx, err.Error(), map[string]interface{}{
				"token": h.authSrv().AddToken(),
			})
		} else {
			h.showForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param, true)
		}
		return
	}

	if formPanel.Responder != nil {
		formPanel.Responder(ctx)
		return
	}

	if ctx.WantJSON() && !param.IsIframe {
		response.OkWithData(ctx, map[string]interface{}{
			"url":   param.PreviousPath,
			"token": h.authSrv().AddToken(),
		})
		return
	}

	if !param.FromList {

		if isNewUrl(param.PreviousPath, param.Prefix) {
			h.showNewForm(ctx, param.Alert, param.Prefix, param.Param.DeleteEditPk().GetRouteParamStr(), true)
			return
		}

		if isEditUrl(param.PreviousPath, param.Prefix) {
			h.showForm(ctx, param.Alert, param.Prefix, param.Param, true, false)
			return
		}

		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>location.href="%s"</script>`, param.PreviousPath))
		ctx.AddHeader(constant.PjaxUrlHeader, param.PreviousPath)
		return
	}

	if param.IsIframe {
		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>
		swal('%s', '', 'success');
		setTimeout(function(){
			$("#%s", window.parent.document).hide();
			$('.modal-backdrop.fade.in', window.parent.document).hide();
		}, 1000)
</script>`, language.Get("success"), param.IframeID))
		return
	}

	buf := h.showTable(ctx, param.Prefix, param.Param.DeletePK().DeleteEditPk(), nil)

	ctx.HTML(http.StatusOK, buf.String())
	ctx.AddHeader(constant.PjaxUrlHeader, param.PreviousPath)
}
```

##### handler.go > GlobalDefenderHandler
``` shell
# GlobalHandlerDefender is a global error handler of admin plugin.
func (h *Handler) GlobalDeferHandler(ctx *context.Context) {
	fmt.Println("Controller")
	logger.Access(ctx)

	if !h.config.OperationLogOff {
		h.RecordOperationLog(ctx)
	}

	if err := recover(); err != nil {
		logger.Error(err)

		var (
			errMsg string
			ok     bool
			e      error
		)

		if errMsg, ok = err.(string); !ok {
			if e, ok = err.(error); ok {
				errMsg = e.Error()
			}
		}

		if errMsg == "" {
			errMsg = "system error"
		}

		if ctx.WantJSON() {
			response.Error(ctx, errMsg)
			return
		}

		if ok, _ = regexp.MatchString("/edit(.*)", ctx.Path()); ok {
			h.setFormWithReturnErrMessage(ctx, errMsg, "edit")
			return
		}
		if ok, _ = regexp.MatchString("/new(.*)", ctx.Path()); ok {
			h.setFormWithReturnErrMessage(ctx, errMsg, "new")
			return
		}

		h.HTML(ctx, auth.Auth(ctx), template.WarningPanelWithDescAndTitle(errMsg, errors.Msg, errors.Msg))
	}
}

```

##### install.go > ShowInstall 
``` shell
# ShowInstall show install page
func (h *Handler) ShowInstall(ctx *context.Context) {

	buffer := new(bytes.Buffer)
	#template.GetInstallPage(buffer)

	#rs, _ := mysql.Query("show tables;")
	#fmt.Println(rs[0]["Tables_in_godmin"])

	#rs2, _ := mysql.Query("show columns from users")
	#fmt.Println(rs2[0]["Field"])

	ctx.HTML(http.StatusOK, buffer.String())
}
```

##### install.go > CheckDatabase
``` shell 
# CheckDatabase check the database connection
func (h *Handler) CheckDatabase(ctx *context.Context) {

	ip := ctx.FormValue("h")
	port := ctx.FormValue("po")
	username := ctx.FormValue("u")
	password := ctx.FormValue("pa")
	databaseName := ctx.FormValue("db")

	SqlDB, err := sql.Open("mysql", username+":"+password+"@tcp("+ip+":"+port+")/"+databaseName+"?charset=utf8mb4")
	if SqlDB != nil {
		if SqlDB.Ping() != nil {
			response.Error(ctx, "请检查参数是否设置正确")
			return
		}
	}

	defer func() {
		_ = SqlDB.Close()
	}()

	if err != nil {
		response.Error(ctx, "请检查参数是否设置正确")
		return

	}

	#db.InitDB(username, password, port, ip, databaseName, 100, 100)

	tables := make([]map[string]interface{}, 0)

	list := "["

	for i := 0; i < len(tables); i++ {
		if i != len(tables)-1 {
			list += `"` + tables[i]["Tables_in_godmin"].(string) + `",`
		} else {
			list += `"` + tables[i]["Tables_in_godmin"].(string) + `"`
		}
	}
	list += "]"

	response.OkWithData(ctx, map[string]interface{}{
		"list": list,
	})
}
```

##### menu.go > ShowMenu
``` shell
# ShowMenu show menu info page
func (h *Handler) ShowMenu(ctx *context.Context) {
	h.getMenuInfoPanel(ctx, "", "")
}
```

##### menu.go > ShowNewMenu
``` shell 
# ShowNewMenu show new menu page
func (h *Handler) ShowNewMenu(ctx *context.Context) {
	h.showNewMenu(ctx, nil)
}
```

##### menu.go > ShowEditMenu
``` shell
# ShowEditMenu show edit menu page
func (h *Handler) ShowEditMenu(ctx *context.Context) {

	plugName := getPlugNameFromReferer(ctx)

	if ctx.Query("id") == "" {
		h.getMenuInfoPanel(ctx, "", template.Get(h.config.Theme).Alert().Warning(errors.WrongID))

		ctx.AddHeader("Content-Type", "text/html; charset=utf-8")
		ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+getMenuPlugNameParams(plugName))
		return
	}

	model := h.table("menu", ctx)
	formInfo, err := model.GetDataWithId(parameter.BaseParam().WithPKs(ctx.Query("id")))

	user := auth.Auth(ctx)

	if err != nil {
		h.HTMLPlug(ctx, user, template.WarningPanelWithDescAndTitle(err.Error(),
			model.GetForm().Description, model.GetForm().Title), plugName)
		return
	}

	h.showEditMenu(ctx, plugName, formInfo, nil)
}
```

##### menu.go > DeleteMenu
``` shell
# DeleteMenu delete the menu of given id
func (h *Handler) DeleteMenu(ctx *context.Context) {
	models.MenuWithId(guard.GetMenuDeleteParam(ctx).Id).SetConn(h.conn).Delete()
	response.OkWithMsg(ctx, language.Get("delete succeed"))
}
```

##### menu.go > EditMenu
``` shell
# EditMenu edit the menu of given id
func (h *Handler) EditMenu(ctx *context.Context) {

	param := guard.GetMenuEditParam(ctx)
	params := getMenuPlugNameParams(param.PluginName)

	if param.HasAlert() {
		h.getMenuInfoPanel(ctx, param.PluginName, param.Alert)
		ctx.AddHeader("Content-Type", "text/html; charset=utf-8")
		ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
		return
	}

	menuModel := models.MenuWithId(param.Id).SetConn(h.conn)

	// TODO: use transaction
	deleteRolesErr := menuModel.DeleteRoles()
	if db.CheckError(deleteRolesErr, db.DELETE) {
		formInfo, _ := h.table("menu", ctx).GetDataWithId(parameter.BaseParam().WithPKs(param.Id))
		h.showEditMenu(ctx, param.PluginName, formInfo, deleteRolesErr)
		ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
		return
	}
	for _, roleId := range param.Roles {
		_, addRoleErr := menuModel.AddRole(roleId)
		if db.CheckError(addRoleErr, db.INSERT) {
			formInfo, _ := h.table("menu", ctx).GetDataWithId(parameter.BaseParam().WithPKs(param.Id))
			h.showEditMenu(ctx, param.PluginName, formInfo, addRoleErr)
			ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
			return
		}
	}

	_, updateErr := menuModel.Update(param.Title, param.Icon, param.Uri, param.Header, param.PluginName, param.ParentId)

	if db.CheckError(updateErr, db.UPDATE) {
		formInfo, _ := h.table("menu", ctx).GetDataWithId(parameter.BaseParam().WithPKs(param.Id))
		h.showEditMenu(ctx, param.PluginName, formInfo, updateErr)
		ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
		return
	}

	h.getMenuInfoPanel(ctx, param.PluginName, "")
	ctx.AddHeader("Content-Type", "text/html; charset=utf-8")
	ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
}
```

##### menu.go > NewMenu
``` shell
func (h *Handler) NewMenu(ctx *context.Context) {

	param := guard.GetMenuNewParam(ctx)
	params := getMenuPlugNameParams(param.PluginName)

	if param.HasAlert() {
		h.getMenuInfoPanel(ctx, param.PluginName, param.Alert)
		ctx.AddHeader("Content-Type", "text/html; charset=utf-8")
		ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
		return
	}

	user := auth.Auth(ctx)

	// TODO: use transaction
	menuModel, createErr := models.Menu().SetConn(h.conn).
		New(param.Title, param.Icon, param.Uri, param.Header, param.PluginName, param.ParentId,
			(menu.GetGlobalMenu(user, h.conn, ctx.Lang(), param.PluginName)).MaxOrder+1)

	if db.CheckError(createErr, db.INSERT) {
		h.showNewMenu(ctx, createErr)
		return
	}

	for _, roleId := range param.Roles {
		_, addRoleErr := menuModel.AddRole(roleId)
		if db.CheckError(addRoleErr, db.INSERT) {
			h.showNewMenu(ctx, addRoleErr)
			return
		}
	}

	menu.GetGlobalMenu(user, h.conn, ctx.Lang(), param.PluginName).AddMaxOrder()

	h.getMenuInfoPanel(ctx, param.PluginName, "")
	ctx.AddHeader("Content-Type", "text/html; charset=utf-8")
	ctx.AddHeader(constant.PjaxUrlHeader, h.routePath("menu")+params)
}
```

##### menu.go > MenuOrder
``` shell 
# MenuOrder change the order of menu items
func (h *Handler) MenuOrder(ctx *context.Context) {

	var data []map[string]interface{}
	_ = json.Unmarshal([]byte(ctx.FormValue("_order")), &data)

	models.Menu().SetConn(h.conn).ResetOrder([]byte(ctx.FormValue("_order")))

	response.Ok(ctx)
}
```

##### new.go > ShowNewForm 
``` shell 
# ShowNewFormnshow a new form page
func (h *Handler) ShowNewForm(ctx *context.Context) {
	param := guard.GetShowNewFormParam(ctx)
	h.showNewForm(ctx, "", param.Prefix, param.Param.GetRouteParamStr(), false)
}
```

##### new.go > NewForm 
``` shell 
# NewForm insert table row into database
func (h *Handler) NewForm(ctx *context.Context) {

	param := guard.GetNewFormParam(ctx)

	// process uploading files, only support local storage
	if len(param.MultiForm.File) > 0 {
		err := file.GetFileEngine(h.config.FileUploadEngine.Name).Upload(param.MultiForm)
		if err != nil {
			logger.Error("get file engine error: ", err)
			if ctx.WantJSON() {
				response.Error(ctx, err.Error())
			} else {
				h.showNewForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param.GetRouteParamStr(), true)
			}
			return
		}
	}

	err := param.Panel.InsertData(param.Value())
	if err != nil {
		logger.Error("insert data error: ", err)
		if ctx.WantJSON() {
			response.Error(ctx, err.Error(), map[string]interface{}{
				"token": h.authSrv().AddToken(),
			})
		} else {
			h.showNewForm(ctx, aAlert().Warning(err.Error()), param.Prefix, param.Param.GetRouteParamStr(), true)
		}
		return
	}

	f := param.Panel.GetActualNewForm()

	if f.Responder != nil {
		f.Responder(ctx)
		return
	}

	if ctx.WantJSON() && !param.IsIframe {
		response.OkWithData(ctx, map[string]interface{}{
			"url":   param.PreviousPath,
			"token": h.authSrv().AddToken(),
		})
		return
	}

	if !param.FromList {

		if isNewUrl(param.PreviousPath, param.Prefix) {
			h.showNewForm(ctx, param.Alert, param.Prefix, param.Param.GetRouteParamStr(), true)
			return
		}

		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>location.href="%s"</script>`, param.PreviousPath))
		ctx.AddHeader(constant.PjaxUrlHeader, param.PreviousPath)
		return
	}

	if param.IsIframe {
		ctx.HTML(http.StatusOK, fmt.Sprintf(`<script>
		swal('%s', '', 'success');
		setTimeout(function(){
			$("#%s", window.parent.document).hide();
			$('.modal-backdrop.fade.in', window.parent.document).hide();
		}, 1000)
</script>`, language.Get("success"), param.IframeID))
		return
	}

	buf := h.showTable(ctx, param.Prefix, param.Param, nil)

	ctx.HTML(http.StatusOK, buf.String())
	ctx.AddHeader(constant.PjaxUrlHeader, h.routePathWithPrefix("info", param.Prefix)+param.Param.GetRouteParamStr())
}
```

##### plugins_tmpl.go > GetPluginPageJS
``` shell
# ...
func GetPluginsPageJS(data PluginsPageJSData) template.JS {
	t := template.New("plugins_page_js").Funcs(map[string]interface{}{
		"lang":     language.Get,
		"plugWord": plugWord,
	})
	t, err := t.Parse(pluginsPageJS)
	if err != nil {
		logger.Error(err)
		return ""
	}
	buf := new(bytes.Buffer)
	err = t.Execute(buf, data)
	if err != nil {
		logger.Error(err)
		return ""
	}
	return template.JS(buf.String())

}
```

##### plugins.go > Plugins 
``` shell
# ...
func (h *Handler) Plugins(ctx *context.Context) {
	list := plugins.Get()
	size := types.Size(6, 3, 2)
	rows := template.HTML("")
	if h.config.IsNotProductionEnvironment() {
		getMoreCover := config.Url("/assets/dist/img/plugin_more.png")
		list = list.Add(plugins.NewBasePluginWithInfoAndIndexURL(plugins.Info{
			Title:     "get more plugins",
			Name:      "",
			MiniCover: getMoreCover,
			Cover:     getMoreCover,
		}, config.Url("/plugins/store"), true))
	}
	for i := 0; i < len(list); i += 6 {
		box1 := aBox().
			SetBody(h.pluginBox(GetPluginBoxParamFromPlug(list[i]))).
			GetContent()
		content := aCol().SetSize(size).SetContent(box1).GetContent()
		offset := len(list) - i
		if offset > 6 {
			offset = 6
		}
		for j := i + 1; j < offset; j++ {
			box2 := aBox().
				SetBody(h.pluginBox(GetPluginBoxParamFromPlug(list[j]))).
				GetContent()
			content += aCol().SetSize(size).SetContent(box2).GetContent()
		}
		rows += aRow().SetContent(content).GetContent()
	}
	h.HTML(ctx, auth.Auth(ctx), types.Panel{
		Content:     rows,
		CSS:         pluginsPageCSS,
		Description: language.GetFromHtml("plugins"),
		Title:       language.GetFromHtml("plugins"),
	})
}
```

##### plugins.go > PluginStore
``` shell
func (h *Handler) PluginStore(ctx *context.Context) {
	var (
		size       = types.Size(12, 6, 4)
		list, page = plugins.GetAll(
			remote_server.GetOnlineReq{
				Page:       ctx.Query("page"),
				Free:       ctx.Query("free"),
				PageSize:   ctx.Query("page_size"),
				Filter:     ctx.Query("filter"),
				Order:      ctx.Query("order"),
				Lang:       h.config.Language,
				Version:    system.Version(),
				CategoryId: ctx.Query("category_id"),
			}, ctx.Cookie(remote_server.TokenKey))
		rows = template.HTML(page.HTML)
	)

	if ctx.Query("page") == "" && len(list) == 0 {
		h.HTML(ctx, auth.Auth(ctx), types.Panel{
			Content: pluginStore404(),
			CSS: template.CSS(`.plugin-store-404-content {
    margin: auto;
    width: 80%;
    text-align: center;
    color: #9e9e9e;
    font-size: 17px;
    height: 250px;
    line-height: 250px;
}`),
			Description: language.GetFromHtml("plugin store"),
			Title:       language.GetFromHtml("plugin store"),
		})
		return
	}

	for i := 0; i < len(list); i += 3 {
		box1 := aBox().
			SetBody(h.pluginStoreBox(GetPluginBoxParamFromPlug(list[i]))).
			GetContent()
		col1 := aCol().SetSize(size).SetContent(box1).GetContent()
		box2, col2, box3, col3 := template.HTML(""), template.HTML(""), template.HTML(""), template.HTML("")
		if i+1 < len(list) {
			box2 = aBox().
				SetBody(h.pluginStoreBox(GetPluginBoxParamFromPlug(list[i+1]))).
				GetContent()
			col2 = aCol().SetSize(size).SetContent(box2).GetContent()
			if i+2 < len(list) {
				box3 = aBox().
					SetBody(h.pluginStoreBox(GetPluginBoxParamFromPlug(list[i+2]))).
					GetContent()
				col3 = aCol().SetSize(size).SetContent(box3).GetContent()
			}
		}
		rows += aRow().SetContent(col1 + col2 + col3).GetContent()
	}

	detailPopupModal := template2.Default().Popup().SetID("detail-popup-modal").
		SetTitle(plugWordHTML("plugin detail")).
		SetBody(pluginsPageDetailPopupBody()).
		SetWidth("730px").
		SetHeight("400px").
		SetFooter("1").
		GetContent()

	buyPopupModal := template2.Default().Popup().SetID("buy-popup-modal").
		SetTitle(plugWordHTML("plugin detail")).
		SetWidth("730px").
		SetHeight("400px").
		SetFooter("1").
		GetContent()

	loginPopupModal := template2.Default().Popup().SetID("login-popup-modal").
		SetTitle(plugWordHTML("login to goadmin member system")).
		SetBody(aForm().SetContent(types.FormFields{
			{Field: "name", Head: plugWord("account"), FormType: form.Text, Editable: true},
			{Field: "password", Head: plugWord("password"), FormType: form.Password, Editable: true,
				HelpMsg: template.HTML(fmt.Sprintf(plugWord("no account? click %s here %s to register."),
					"<a target='_blank' href='http://www.go-admin.cn/register'>", "</a>"))},
		}).GetContent()).
		SetWidth("540px").
		SetHeight("250px").
		SetFooterHTML(template.HTML(`<button type="button" class="btn btn-primary" onclick="login()">` +
			plugWord("login") + `</button>`)).
		GetContent()

	h.HTML(ctx, auth.Auth(ctx), types.Panel{
		Content:     rows + detailPopupModal + buyPopupModal + loginPopupModal,
		CSS:         pluginsStorePageCSS + template.CSS(page.CSS),
		JS:          template.JS(page.JS) + GetPluginsPageJS(PluginsPageJSData{Prefix: h.config.Prefix()}),
		Description: language.GetFromHtml("plugin store"),
		Title:       language.GetFromHtml("plugin store"),
	})
}
```

##### plugins.go > PluginDetail
``` shell
func (h *Handler) PluginDetail(ctx *context.Context) {

	name := ctx.Query("name")

	plug, exist := plugins.FindByNameAll(name)
	if !exist {
		ctx.JSON(http.StatusOK, gin.H{
			"code": 400,
			"msg":  "bad request",
		})
		return
	}

	info := plug.GetInfo()

	if info.MiniCover == "" {
		info.MiniCover = config.Url("/assets/dist/img/plugin_default.png")
	}

	ctx.JSON(http.StatusOK, gin.H{
		"code": 0,
		"msg":  "ok",
		"data": gin.H{
			"mini_cover":      info.MiniCover,
			"title":           language.GetWithScope(info.Title, name),
			"author":          fmt.Sprintf(plugWord("provided by %s"), language.GetWithScope(info.Author, name)),
			"introduction":    language.GetWithScope(info.Description, name),
			"website":         language.GetWithScope(info.Website, name),
			"version":         language.GetWithScope(info.Version, name),
			"created_at":      language.GetWithScope(info.CreateDate.Format("2006-01-02"), name),
			"updated_at":      language.GetWithScope(info.UpdateDate.Format("2006-01-02"), name),
			"downloaded":      info.Downloaded,
			"download_reboot": plugins.Exist(plug),
			"skip":            info.SkipInstallation,
			"uuid":            info.Uuid,
			"upgrade":         info.CanUpdate,
			"install":         plug.IsInstalled(),
			"free":            info.IsFree(),
		},
	})
}
```

##### plugins.go > GetPluginParamFromPlug
``` shell
func GetPluginBoxParamFromPlug(plug plugins.Plugin) PluginBoxParam {
	return PluginBoxParam{
		Info:           plug.GetInfo(),
		Install:        plug.IsInstalled(),
		Upgrade:        plug.GetInfo().CanUpdate,
		Skip:           plug.GetInfo().SkipInstallation,
		DownloadReboot: plugins.Exist(plug),
		Name:           plug.Name(),
		IndexURL:       plug.GetIndexURL(),
	}
}
```

##### plugins.go > PluginDownload
``` shell
# ...
func (h *Handler) PluginDownload(ctx *context.Context) {

	if !h.config.Debug {
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 400,
			"msg":  plugWord("change to debug mode first"),
		})
		return
	}

	name := ctx.FormValue("name")

	if name == "" {
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 400,
			"msg":  plugWord("download fail, wrong name"),
		})
		return
	}

	plug, exist := plugins.FindByNameAll(name)

	if !exist {
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 400,
			"msg":  plugWord("download fail, plugin not exist"),
		})
		return
	}

	if !plug.GetInfo().IsFree() && !plug.GetInfo().HasBought {
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 400,
			"msg":  plugWord("download fail, plugin has not been bought"),
		})
		return
	}

	downloadURL := plug.GetInfo().Url
	extraDownloadURL := plug.GetInfo().ExtraDownloadUrl

	if !plug.GetInfo().IsFree() {
		var err error
		downloadURL, extraDownloadURL, err = remote_server.GetDownloadURL(plug.GetInfo().Uuid, ctx.Cookie(remote_server.TokenKey))
		if err != nil {
			logger.Error("download plugins error", err)
			ctx.JSON(http.StatusOK, map[string]interface{}{
				"code": 500,
				"msg":  plugWord("download fail"),
			})
			return
		}
	}

	tempFile := "./temp-" + utils.Uuid(10) + ".zip"

	err := utils.DownloadTo(downloadURL, tempFile)

	if err != nil {
		logger.Error("download plugins error", map[string]interface{}{
			"error":       err,
			"downloadURL": downloadURL,
		})
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 500,
			"msg":  plugWord("download fail"),
		})
		return
	}

	gopath := os.Getenv("GOPATH")

	if gopath == "" {
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 500,
			"msg":  plugWord("golang develop environment does not exist"),
		})
		return
	}

	gomodule := os.Getenv("GO111MODULE")
	base := filepath.Dir(plug.GetInfo().ModulePath)
	installPath := ""

	if gomodule == "off" {
		installPath = filepath.ToSlash(gopath + "/src/" + base)
	} else {
		installPath = filepath.ToSlash(gopath + "/pkg/mod/" + base)
	}

	err = utils.UnzipDir(tempFile, installPath)

	if err != nil {
		logger.Error("download plugins, unzip error", map[string]interface{}{
			"error":       err,
			"installPath": installPath,
		})
		ctx.JSON(http.StatusOK, map[string]interface{}{
			"code": 500,
			"msg":  plugWord("download fail"),
		})
		return
	}

	_ = os.Remove(tempFile)

	if len(downloadURL) > 18 && downloadURL[:18] == "https://github.com" {
		name := filepath.Base(plug.GetInfo().ModulePath)
		version := strings.ReplaceAll(plug.GetInfo().Version, "v", "")
		rawPath := installPath + "/" + name
		nowPath := rawPath + "-" + version
		if gomodule == "off" {
			err = os.Rename(nowPath, rawPath)
		} else {
			err = os.Rename(nowPath, rawPath+"@"+plug.GetInfo().Version)
		}
		if err != nil {
			logger.Error("download plugins, rename error", map[string]interface{}{
				"error":   err,
				"nowPath": nowPath,
				"rawPath": rawPath,
			})
			ctx.JSON(http.StatusOK, map[string]interface{}{
				"code": 500,
				"msg":  plugWord("download fail"),
			})
			return
		}
	} else if gomodule != "off" {
		rawPath := installPath + "/" + name
		err = os.Rename(rawPath, rawPath+"@"+plug.GetInfo().Version)
		if err != nil {
			logger.Error("download plugins, rename error", map[string]interface{}{
				"error":   err,
				"rawPath": rawPath,
			})
			ctx.JSON(http.StatusOK, map[string]interface{}{
				"code": 500,
				"msg":  plugWord("download fail"),
			})
			return
		}
	}

	if h.config.BootstrapFilePath != "" && utils.FileExist(h.config.BootstrapFilePath) {
		content, err := ioutil.ReadFile(h.config.BootstrapFilePath)
		if err != nil {
			logger.Error("read bootstrap file error: ", err)
		} else {
			err = ioutil.WriteFile(h.config.BootstrapFilePath, []byte(string(content)+`
import _ "`+plug.GetInfo().ModulePath+`"`), 0644)
			if err != nil {
				logger.Error("write bootstrap file error: ", err)
			}
		}
	}

	if h.config.GoModFilePath != "" && utils.FileExist(h.config.GoModFilePath) &&
		plug.GetInfo().CanUpdate && plug.GetInfo().OldVersion != "" {
		content, _ := ioutil.ReadFile(h.config.BootstrapFilePath)
		src := plug.GetInfo().ModulePath + " " + plug.GetInfo().OldVersion
		dist := plug.GetInfo().ModulePath + " " + plug.GetInfo().Version
		content = bytes.ReplaceAll(content, []byte(src), []byte(dist))
		_ = ioutil.WriteFile(h.config.BootstrapFilePath, content, 0644)
	}

	// TODO: 实现运行环境与编译环境隔离

	if plug.GetInfo().ExtraDownloadUrl != "" {
		err = utils.DownloadTo(extraDownloadURL, "./"+plug.Name()+"_extra_"+
			fmt.Sprintf("%d", time.Now().Unix())+".zip")
		if err != nil {
			logger.Error("failed to download "+plug.Name()+" extra data: ", err)
		}
	}

	plug.(*plugins.BasePlugin).Info.Downloaded = true
	plug.(*plugins.BasePlugin).Info.CanUpdate = false

	ctx.JSON(http.StatusOK, map[string]interface{}{
		"code": 0,
		"msg":  plugWord("download success, restart to install"),
	})
}
```

##### plugins.go > ServerLogin
``` shell
func (h *Handler) ServerLogin(ctx *context.Context) {
	param := guard.GetServerLoginParam(ctx)
	res := remote_server.Login(param.Account, param.Password)
	if res.Code == 0 && res.Data.Token != "" {
		ctx.SetCookie(&http.Cookie{
			Name:     remote_server.TokenKey,
			Value:    res.Data.Token,
			Expires:  time.Now().Add(time.Second * time.Duration(res.Data.Expire/1000)),
			HttpOnly: true,
			Path:     "/",
		})
	}
	ctx.JSON(http.StatusOK, gin.H{
		"code": res.Code,
		"data": res.Data,
		"msg":  res.Msg,
	})
}
```

##### show.go > ShowInfo 
``` shell
# ShowInfo show info page
func (h *Handler) ShowInfo(ctx *context.Context) {

	prefix := ctx.Query(constant.PrefixKey)

	panel := h.table(prefix, ctx)

	if panel.GetOnlyUpdateForm() {
		ctx.Redirect(h.routePathWithPrefix("show_edit", prefix))
		return
	}

	if panel.GetOnlyNewForm() {
		ctx.Redirect(h.routePathWithPrefix("show_new", prefix))
		return
	}

	if panel.GetOnlyDetail() {
		ctx.Redirect(h.routePathWithPrefix("detail", prefix))
		return
	}

	params := parameter.GetParam(ctx.Request.URL, panel.GetInfo().DefaultPageSize, panel.GetInfo().SortField,
		panel.GetInfo().GetSort())

	buf := h.showTable(ctx, prefix, params, panel)
	ctx.HTML(http.StatusOK, buf.String())
}
```

##### show.go > Assets 
``` shell 
# Assets return frontend assets accoring the request path.
func (h *Handler) Assets(ctx *context.Context) {
	filepath := h.config.URLRemovePrefix(ctx.Path())
	data, err := aTemplate().GetAsset(filepath)

	if err != nil {
		data, err = template.GetAsset(filepath)
		if err != nil {
			logger.Error("asset err", err)
			ctx.Write(http.StatusNotFound, map[string]string{}, "")
			return
		}
	}

	var contentType = mime.TypeByExtension(path.Ext(filepath))

	if contentType == "" {
		contentType = "application/octet-stream"
	}

	etag := fmt.Sprintf("%x", md5.Sum(data))

	if match := ctx.Headers("If-None-Match"); match != "" {
		if strings.Contains(match, etag) {
			ctx.SetStatusCode(http.StatusNotModified)
			return
		}
	}

	ctx.DataWithHeaders(http.StatusOK, map[string]string{
		"Content-Type":   contentType,
		"Cache-Control":  "max-age=2592000",
		"Content-Length": strconv.Itoa(len(data)),
		"ETag":           etag,
	}, data)
}
```

##### show.go > Export
``` shell 
# Export export table rows as exel object
func (h *Handler) Export(ctx *context.Context) {
	param := guard.GetExportParam(ctx)

	tableName := "Sheet1"
	prefix := ctx.Query(constant.PrefixKey)
	panel := h.table(prefix, ctx)

	f := excelize.NewFile()
	index := f.NewSheet(tableName)
	f.SetActiveSheet(index)

	var (
		infoData  table.PanelInfo
		fileName  string
		err       error
		tableInfo = panel.GetInfo()
		params    parameter.Parameters
	)

	if fn := panel.GetInfo().ExportProcessFn; fn != nil {
		params = parameter.GetParam(ctx.Request.URL, tableInfo.DefaultPageSize, tableInfo.SortField,
			tableInfo.GetSort())
		p, err := fn(params.WithIsAll(param.IsAll))
		if err != nil {
			response.Error(ctx, "export error")
			return
		}
		infoData.Thead = p.Thead
		infoData.InfoList = p.InfoList
	} else {
		if len(param.Id) == 0 {
			params = parameter.GetParam(ctx.Request.URL, tableInfo.DefaultPageSize, tableInfo.SortField,
				tableInfo.GetSort())
			infoData, err = panel.GetData(params.WithIsAll(param.IsAll))
			fileName = fmt.Sprintf("%s-%d-page-%s-pageSize-%s.xlsx", tableInfo.Title, time.Now().Unix(),
				params.Page, params.PageSize)
		} else {
			infoData, err = panel.GetDataWithIds(parameter.GetParam(ctx.Request.URL,
				tableInfo.DefaultPageSize, tableInfo.SortField, tableInfo.GetSort()).WithPKs(param.Id...))
			fileName = fmt.Sprintf("%s-%d-id-%s.xlsx", tableInfo.Title, time.Now().Unix(), strings.Join(param.Id, "_"))
		}
		if err != nil {
			response.Error(ctx, "export error")
			return
		}
	}

	// TODO: support any numbers of fields.
	orders := []string{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K",
		"L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"}

	if len(infoData.Thead) > 26 {
		j := -1
		for i := 0; i < len(infoData.Thead)-26; i++ {
			if i%26 == 0 {
				j++
			}
			letter := orders[j] + orders[i%26]
			orders = append(orders, letter)
		}
	}

	columnIndex := 0
	for _, head := range infoData.Thead {
		if !head.Hide {
			f.SetCellValue(tableName, orders[columnIndex]+"1", head.Head)
			columnIndex++
		}
	}

	count := 2
	for _, info := range infoData.InfoList {
		columnIndex = 0
		for _, head := range infoData.Thead {
			if !head.Hide {
				if tableInfo.IsExportValue() {
					f.SetCellValue(tableName, orders[columnIndex]+strconv.Itoa(count), info[head.Field].Value)
				} else {
					f.SetCellValue(tableName, orders[columnIndex]+strconv.Itoa(count), info[head.Field].Content)
				}
				columnIndex++
			}
		}
		count++
	}

	buf, err := f.WriteToBuffer()

	if err != nil || buf == nil {
		response.Error(ctx, "export error")
		return
	}

	ctx.AddHeader("content-disposition", `attachment; filename=`+fileName)
	ctx.Data(200, "application/vnd.ms-excel", buf.Bytes())
}
```

##### system.go > SystemInfo
``` shell 
#...
func (h *Handler) SystemInfo(ctx *context.Context) {

	size := types.Size(6, 6, 6)

	box1 := aBox().
		WithHeadBorder().
		SetHeader("<b>" + lg("application") + "</b>").
		SetBody(stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("app_name")},
				"value": types.InfoItem{Content: "GoAdmin"},
			}, {
				"key":   types.InfoItem{Content: lg("go_admin_version")},
				"value": types.InfoItem{Content: template.HTML(system.Version())},
			}, {
				"key":   types.InfoItem{Content: lg("theme_name")},
				"value": types.InfoItem{Content: template.HTML(aTemplate().Name())},
			}, {
				"key":   types.InfoItem{Content: lg("theme_version")},
				"value": types.InfoItem{Content: template.HTML(aTemplate().GetVersion())},
			},
		})).
		GetContent()

	app := system.GetAppStatus()

	box2 := aBox().
		WithHeadBorder().
		SetHeader("<b>" + lg("application run") + "</b>").
		SetBody(stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("current_heap_usage")},
				"value": types.InfoItem{Content: template.HTML(app.HeapAlloc)},
			},
			{
				"key":   types.InfoItem{Content: lg("heap_memory_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.HeapSys)},
			},
			{
				"key":   types.InfoItem{Content: lg("heap_memory_idle")},
				"value": types.InfoItem{Content: template.HTML(app.HeapIdle)},
			},
			{
				"key":   types.InfoItem{Content: lg("heap_memory_in_use")},
				"value": types.InfoItem{Content: template.HTML(app.HeapInuse)},
			},
			{
				"key":   types.InfoItem{Content: lg("heap_memory_released")},
				"value": types.InfoItem{Content: template.HTML(app.HeapReleased)},
			},
			{
				"key":   types.InfoItem{Content: lg("heap_objects")},
				"value": types.InfoItem{Content: itos(app.HeapObjects)},
			},
		}) + `<div><hr></div>` + stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("next_gc_recycle")},
				"value": types.InfoItem{Content: template.HTML(app.NextGC)},
			}, {
				"key":   types.InfoItem{Content: lg("last_gc_time")},
				"value": types.InfoItem{Content: template.HTML(app.LastGC)},
			}, {
				"key":   types.InfoItem{Content: lg("total_gc_pause")},
				"value": types.InfoItem{Content: template.HTML(app.PauseTotalNs)},
			}, {
				"key":   types.InfoItem{Content: lg("last_gc_pause")},
				"value": types.InfoItem{Content: template.HTML(app.PauseNs)},
			}, {
				"key":   types.InfoItem{Content: lg("gc_times")},
				"value": types.InfoItem{Content: itos(app.NumGC)},
			},
		})).
		GetContent()

	col1 := aCol().SetSize(size).SetContent(box1 + box2).GetContent()

	box4 := aBox().
		WithHeadBorder().
		SetHeader("<b>" + lg("application run") + "</b>").
		SetBody(stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("golang_version")},
				"value": types.InfoItem{Content: template.HTML(runtime.Version())},
			}, {
				"key":   types.InfoItem{Content: lg("process_id")},
				"value": types.InfoItem{Content: itos(os.Getpid())},
			}, {
				"key":   types.InfoItem{Content: lg("server_uptime")},
				"value": types.InfoItem{Content: template.HTML(app.Uptime)},
			}, {
				"key":   types.InfoItem{Content: lg("current_goroutine")},
				"value": types.InfoItem{Content: itos(app.NumGoroutine)},
			},
		}) + `<div><hr></div>` + stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("current_memory_usage")},
				"value": types.InfoItem{Content: template.HTML(app.MemAllocated)},
			}, {
				"key":   types.InfoItem{Content: lg("total_memory_allocated")},
				"value": types.InfoItem{Content: template.HTML(app.MemTotal)},
			}, {
				"key":   types.InfoItem{Content: lg("memory_obtained")},
				"value": types.InfoItem{Content: itos(app.MemSys)},
			}, {
				"key":   types.InfoItem{Content: lg("pointer_lookup_times")},
				"value": types.InfoItem{Content: itos(app.Lookups)},
			}, {
				"key":   types.InfoItem{Content: lg("memory_allocate_times")},
				"value": types.InfoItem{Content: itos(app.MemMallocs)},
			}, {
				"key":   types.InfoItem{Content: lg("memory_free_times")},
				"value": types.InfoItem{Content: itos(app.MemFrees)},
			},
		}) + `<div><hr></div>` + stripedTable([]map[string]types.InfoItem{
			{
				"key":   types.InfoItem{Content: lg("bootstrap_stack_usage")},
				"value": types.InfoItem{Content: template.HTML(app.StackInuse)},
			}, {
				"key":   types.InfoItem{Content: lg("stack_memory_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.StackSys)},
			}, {
				"key":   types.InfoItem{Content: lg("mspan_structures_usage")},
				"value": types.InfoItem{Content: template.HTML(app.MSpanInuse)},
			}, {
				"key":   types.InfoItem{Content: lg("mspan_structures_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.HeapSys)},
			}, {
				"key":   types.InfoItem{Content: lg("mcache_structures_usage")},
				"value": types.InfoItem{Content: template.HTML(app.MCacheInuse)},
			}, {
				"key":   types.InfoItem{Content: lg("mcache_structures_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.MCacheSys)},
			}, {
				"key":   types.InfoItem{Content: lg("profiling_bucket_hash_table_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.BuckHashSys)},
			}, {
				"key":   types.InfoItem{Content: lg("gc_metadata_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.GCSys)},
			}, {
				"key":   types.InfoItem{Content: lg("other_system_allocation_obtained")},
				"value": types.InfoItem{Content: template.HTML(app.OtherSys)},
			},
		})).
		GetContent()

	col2 := aCol().SetSize(size).SetContent(box4).GetContent()

	row := aRow().SetContent(col1 + col2).GetContent()

	h.HTML(ctx, auth.Auth(ctx), types.Panel{
		Content:     row,
		Description: language.GetFromHtml("system info", "system"),
		Title:       language.GetFromHtml("system info", "system"),
	})
}
```

##### update.go > Update
``` shell
# Update update the table row of given id
func (h *Handler) Update(ctx *context.Context) {

	param := guard.GetUpdateParam(ctx)

	err := param.Panel.UpdateData(param.Value)

	if err != nil {
		response.Error(ctx, err.Error())
		return
	}

	response.Ok(ctx)
}
```

##### operation.go > Operation
``` shell 
func (h *Handler) Operation(ctx *context.Context) {
	id := ctx.Query("__goadmin_op_id")
	if !h.OperationHandler(config.Url("/operation/"+id), ctx) {
		errMsg := "not found"
		if ctx.Headers(constant.PjaxHeader) == "" && ctx.Method() != "GET" {
			response.BadRequest(ctx, errMsg)
		} else {
			response.Alert(ctx, errMsg, errMsg, errMsg, h.conn, h.navButtons)
		}
		return
	}
}
```

##### operation.go > RecordOperationLog
``` shell 
# RecordOpertationLog record all operation logs, store into database
func (h *Handler) RecordOperationLog(ctx *context.Context) {
	if user, ok := ctx.UserValue["user"].(models.UserModel); ok {
		var input []byte
		form := ctx.Request.MultipartForm
		if form != nil {
			input, _ = json.Marshal((*form).Value)
		}

		models.OperationLog().SetConn(h.conn).New(user.Id, ctx.Path(), ctx.Method(), ctx.LocalIP(), string(input))
	}
}
```

##### common_test.go > TestIsInfoUrl 
``` shell 
func TestIsInfoUrl(t *testing.T) {
	u := "https://localhost:8098/admin/info/user?id=sdfs"
	assert.Equal(t, true, isInfoUrl(u))
}
```

##### common_test.go > TestIsNewUrl 
``` shell 
func TestIsNewUrl(t *testing.T) {
	u := "https://localhost:8098/admin/info/user/new?id=sdfs"
	assert.Equal(t, true, isNewUrl(u, "user"))
}

```

---------------

### Databases